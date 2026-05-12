# 编码风格详情

> 触发条件：任何涉及 STM32 C 代码新增、修改、重构、HAL 调用、主循环调度、超时等待、看门狗或断言机制的任务，必须先阅读本文件。
> 此文件为按需加载的详细参考，核心规则见 `AGENTS.md`。

## 非阻塞定时模式

`while(1)` 中只调用高层接口，不写复杂业务逻辑。周期任务统一使用 `HAL_GetTick()` 做非阻塞调度。

```c
static uint32_t last_tick = 0;
uint32_t now = HAL_GetTick();

if ((now - last_tick) >= 1000U)
{
    last_tick = now;
    dht11_process();
}
```

## 通信超时模板

允许短时间等待硬件状态，但必须带超时退出，禁止无限 `while`。超时后必须走统一错误处理流程，详见 `error_handling_spec.md`。

```c
int8_t wait_transfer_done(uint32_t timeout_ms)
{
    uint32_t start_tick = HAL_GetTick();

    while (transfer_complete == 0U)
    {
        if ((HAL_GetTick() - start_tick) >= timeout_ms)
        {
            error_report(ERR_TIMEOUT);
            return -1;
        }
    }

    return 0;
}
```

## HAL 返回值检查

阻塞式 HAL 函数必须检查返回值，不允许忽略通信、写入、发送、接收结果。

```c
if (HAL_OK != HAL_I2C_Mem_Read(&hi2c1, addr, reg, I2C_MEMADD_SIZE_8BIT, buf, len, 100U))
{
    error_report(ERR_COMM);
    return -1;
}
```

## 微秒级时序规则

DHT11、DS18B20、1-Wire、超声波触发脉冲等微秒级时序，不允许使用 `HAL_Delay()`。

按架构选择微秒延时实现方案：

| MCU 内核 | 推荐方案 | 说明 |
|----------|----------|------|
| Cortex-M3 / M4 / M7（F1/F4/F7/H7/L4） | DWT CYCCNT | 先 `CoreDebug->DEMCR \|= CoreDebug_DEMCR_TRCENA_Msk;` `DWT->CTRL \|= DWT_CTRL_CYCCNTENA_Msk;` 再用 CYCCNT 差值计算微秒 |
| Cortex-M0 / M0+（F0/G0/L0） | 专用 TIM 计数 | 这些内核**没有 DWT**，必须用一个空闲 TIM 配成 1MHz（1μs 递增），`__HAL_TIM_GET_COUNTER()` 做 busy-wait 基准 |
| 兜底方案 | `__NOP()` 循环 | 仅用于极简短等待且无 TIM 资源时，必须在目标主频下实测校准，并在注释中写明 |

共用约束：

* 微秒等待函数必须有最大等待边界，不能形成不可退出死循环。
* 若微秒级等待会短时间关闭中断，必须说明影响范围，并避免影响 UART 接收、系统节拍和 RTOS 调度。
* 切换 MCU 系列或主频时，必须重新核对微秒延时实现是否仍然有效。

DWT 方案示例（仅限 M3/M4/M7）：

```c
#if defined(DWT) && defined(CoreDebug)

static void dwt_init(void)
{
    CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;
    DWT->CYCCNT = 0U;
    DWT->CTRL |= DWT_CTRL_CYCCNTENA_Msk;
}

void delay_us(uint32_t us)
{
    uint32_t start = DWT->CYCCNT;
    uint32_t ticks = us * (SystemCoreClock / 1000000U);
    while ((DWT->CYCCNT - start) < ticks) { /* busy wait */ }
}

#endif
```

`us` 必须满足 `us <= UINT32_MAX / (SystemCoreClock / 1000000U)`，避免 `ticks` 乘法溢出。需要更长等待时使用毫秒级非阻塞调度，不要用微秒 busy-wait。

TIM 方案示例（M0/M0+ 必选）：

```c
/* 假设 htim14 已在 CubeMX 中配置为 1MHz 计数频率、65535 自动重装 */
static void delay_us_chunk(uint16_t us)
{
    uint16_t start = (uint16_t)__HAL_TIM_GET_COUNTER(&htim14);
    while ((uint16_t)((uint16_t)__HAL_TIM_GET_COUNTER(&htim14) - start) < us) { /* busy wait */ }
}

void delay_us(uint32_t us)
{
    while (us >= 32768U)
    {
        delay_us_chunk(32767U);
        us -= 32767U;
    }

    if (us > 0U)
    {
        delay_us_chunk((uint16_t)us);
    }
}
```

16-bit TIM 单次等待上限必须小于 32768us；超过时按上例分段，保证回绕差值判断可靠。

不确定芯片是否支持 DWT 时，必须向张三确认，不要凭记忆写 `DWT->CYCCNT`。

## 看门狗喂狗规范

启用 IWDG/WWDG 的项目，喂狗只能放在主循环高层任务完成一次调度之后。

```c
while (1)
{
    app_process();
    ui_process();
    esp_comm_process();

    if (HAL_OK != HAL_IWDG_Refresh(&hiwdg))
    {
        error_report(ERR_WATCHDOG);
    }
}
```

禁止在以下位置喂狗：

* 通信等待循环内部
* 传感器等待响应循环内部
* 错误恢复死等逻辑内部
* 中断回调内部

这样可以确保系统一旦卡在某个任务中，看门狗能够真正复位系统。

## Debug-only ASSERT 宏

断言只用于开发阶段暴露程序员错误，不能替代运行时错误处理。Release 模式下断言为空操作。

```c
#ifdef DEBUG
#define APP_ASSERT(expr)                         \
    do                                           \
    {                                            \
        if (!(expr))                             \
        {                                        \
            __BKPT(0);                           \
        }                                        \
    } while (0)
#else
#define APP_ASSERT(expr) ((void)0)
#endif
```

使用规则：

* 指针、数组长度、枚举范围等内部不变量可用 `APP_ASSERT()`。
* 通信失败、传感器离线、用户输入越界必须用错误处理和范围钳位，不得只写断言。

## stdint.h 类型使用场景

| 类型 | 范围 | 典型用途 |
|------|------|---------|
| `uint8_t` | 0~255 | 状态标志、GPIO 电平、传感器状态 |
| `int16_t` | -32768~32767 | 小范围计数、带符号传感器数据 |
| `uint16_t` | 0~65535 | ADC 数值、按键计数 |
| `uint32_t` | 0~4294967295 | 时间戳、累计计数、超时判断 |

## 指针安全

```c
uint8_t *ptr = NULL;

ptr = get_buffer();
if (ptr == NULL)
{
    error_report(ERR_PARAM);
    return -1;
}
```

## volatile 使用场景

ISR 中设置、主循环中读取的变量必须加 `volatile`。

```c
static volatile uint8_t uart_rx_flag = 0;

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart == &huart1)
    {
        uart_rx_flag = 1U;
    }
}
```

## 文件结构示例

```
Core/
├── Inc/
│   ├── dht11.h
│   ├── ui.h
│   └── esp_comm.h
├── Src/
│   ├── main.c
│   ├── dht11.c
│   ├── ui.c
│   └── esp_comm.c
```

`main.c` 的 `while(1)` 中只调用高层接口：

```c
/* USER CODE BEGIN WHILE */
while (1)
{
    app_process();
    ui_process();
    esp_comm_process();
    /* USER CODE END WHILE */
}
```
