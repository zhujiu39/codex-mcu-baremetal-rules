# 错误处理规范

> 触发条件：任何涉及 HAL 返回值处理、通信失败、传感器离线、参数非法、错误提示、降级运行、复位策略或看门狗配合的任务，必须先阅读本文件。
> 本文件定义全局错误码、错误等级、上报方式和严重错误处理边界。

## 1. 错误码定义

项目自定义模块统一使用 `ErrCode` 返回错误状态。底层驱动如需返回 `int8_t`，也必须在注释中映射到本错误码。

```c
typedef enum
{
    ERR_OK = 0,
    ERR_PARAM,
    ERR_TIMEOUT,
    ERR_COMM,
    ERR_CRC,
    ERR_SENSOR_OFFLINE,
    ERR_OVER_RANGE,
    ERR_STORAGE,
    ERR_WATCHDOG,
    ERR_FATAL
} ErrCode;
```

含义：

| 错误码 | 含义 | 典型场景 |
|--------|------|----------|
| `ERR_OK` | 正常 | 操作成功 |
| `ERR_PARAM` | 参数错误 | 空指针、长度越界、非法枚举 |
| `ERR_TIMEOUT` | 超时 | I2C/SPI/UART 等待完成超时 |
| `ERR_COMM` | 通信失败 | HAL 通信返回非 `HAL_OK` |
| `ERR_CRC` | 校验失败 | 传感器 CRC、Flash CRC、可选协议校验；默认 JSON Lines 协议不使用 CRC |
| `ERR_SENSOR_OFFLINE` | 传感器离线 | 无响应、初始化失败 |
| `ERR_OVER_RANGE` | 数据越界 | 采样值超出可信范围 |
| `ERR_STORAGE` | 存储错误 | Flash/EEPROM 读写失败 |
| `ERR_WATCHDOG` | 看门狗相关错误 | 喂狗失败或复位原因记录 |
| `ERR_FATAL` | 严重错误 | 关键外设不可恢复 |

## 2. 错误等级

错误处理分三级，不同等级采用不同恢复策略。

| 等级 | 处理策略 | 示例 |
|------|----------|------|
| 普通错误 | 记录错误计数，保留上一次有效值，系统继续运行 | 单次传感器读数越界 |
| 通信错误 | 计数并按模块规则重试，连续超过阈值后降级 | ESP01S ACK 超时 |
| 严重错误 | 连续失败达到 fatal 阈值后进入降级或等待看门狗复位 | 系统关键通信持续失败 |

## 3. 错误上报方式

错误上报必须尽量不阻塞主循环。

* 串口日志：用于调试和 ESP01S 诊断，输出必须限速。
* OLED 提示：只显示当前最重要错误，避免刷屏。
* LED 闪烁：适合无屏项目，闪烁模式必须非阻塞。
* 蜂鸣器：只用于短促提示，不用于长时间报警。

建议错误上报接口：

```c
void error_report(ErrCode code);
ErrCode error_get_last(void);
void error_clear(ErrCode code);
uint16_t error_get_count(ErrCode code);
uint8_t error_is_degraded(ErrCode code);
```

## 4. 错误计数器与连续失败阈值

模块级错误必须支持累计计数和连续失败计数。单次失败只记录，不应立即触发降级。

```c
typedef struct
{
    ErrCode last_code;
    uint16_t total_count;
    uint8_t consecutive_count;
    uint8_t degraded;
} ErrorState;
```

默认阈值：

```c
#define ERROR_DEGRADE_COUNT       3U
#define ERROR_FATAL_COUNT         10U
#define ERROR_COUNT_MAX           65535U
```

处理规则：

* `total_count` 记录累计错误次数，到 `ERROR_COUNT_MAX` 后保持饱和，不回绕。
* `consecutive_count` 记录连续失败次数，单次成功后必须清零。
* 达到 `ERROR_DEGRADE_COUNT` 后进入降级运行。
* 达到 `ERROR_FATAL_COUNT` 后才允许进入严重错误处理。
* 通信类错误可结合 `comm_protocol_spec.md` 中的重传次数；重传全部失败后记一次连续失败。
* 不同模块应有独立计数器，禁止所有错误混用一个全局连续失败计数。

示例：

```c
void error_record(ErrorState *state, ErrCode code)
{
    if (state == NULL)
    {
        return;
    }

    state->last_code = code;

    if (state->total_count < ERROR_COUNT_MAX)
    {
        state->total_count++;
    }

    if (state->consecutive_count < 255U)
    {
        state->consecutive_count++;
    }

    if (state->consecutive_count >= ERROR_DEGRADE_COUNT)
    {
        state->degraded = 1U;
    }
}

void error_record_success(ErrorState *state)
{
    if (state == NULL)
    {
        return;
    }

    state->consecutive_count = 0U;
    state->degraded = 0U;
}
```

## 5. 降级运行规则

当某个模块不可用时，优先降级运行，不直接让整机停摆。

* 传感器离线：显示离线状态，控制逻辑使用安全默认值或保留上一次有效值。
* 云端通信失败：本地 UI 和自动控制继续运行，通信模块进入重试节流。
* OLED 失败：核心控制逻辑继续运行，串口或 LED 输出错误状态。
* 存储失败：使用默认参数运行，并提示参数未保存。

降级退出条件：

* 传感器类错误：连续 1~3 次成功读取后退出降级，具体次数按项目风险确定。
* 通信类错误：收到合法 ACK 或连续成功通信后退出降级。
* 存储类错误：重新写入成功后退出降级。

## 6. 严重错误与看门狗边界

严重错误不允许在原地死循环等待。若需要复位，必须让看门狗完成复位，且禁止在 fatal 状态下喂狗。

```c
static volatile uint8_t error_fatal_active = 0U;

void error_enter_fatal(ErrCode code)
{
    error_report(code);
    error_fatal_active = 1U;
}

void error_fatal_process(void)
{
    if (error_fatal_active != 0U)
    {
        error_led_process();
        /* Do not feed watchdog in fatal state. */
    }
}
```

如果项目未启用看门狗，严重错误只能进入降级模式或保持错误提示，不得写不可退出死循环。

## 7. 复位源诊断（强制）

启用 IWDG/WWDG 或需要事后复盘的项目，`main()` 进入 USER CODE 区后必须**第一时间**读取 `RCC->CSR`，解析复位原因并记录，然后清除标志位。不记录就复位的系统在现场排障时等同黑盒。

解析规则：

| `RCC->CSR` 标志 | 复位源 | 处理 |
|-----------------|--------|------|
| `RCC_FLAG_IWDGRST` | 独立看门狗复位 | 记录 `ERR_WATCHDOG`，表示上次存在卡死 |
| `RCC_FLAG_WWDGRST` | 窗口看门狗复位 | 记录 `ERR_WATCHDOG`，表示上次存在超时 |
| `RCC_FLAG_SFTRST` | 软件复位 | 正常场景（OTA、手动复位），不计入错误 |
| `RCC_FLAG_PORRST` / `RCC_FLAG_BORRST` | 上电 / 掉电复位 | 首次上电，无需告警 |
| `RCC_FLAG_PINRST` | 外部 NRST 复位 | 调试重启，无需告警 |
| `RCC_FLAG_LPWRRST` | 低功耗复位 | 按项目决定是否告警 |

不同 STM32 系列宏名略有差异（F0 为 `RCC_FLAG_IWDGRST`，部分系列写作 `RCC_FLAG_IWDG1RST`），以对应 HAL 头文件为准。

参考实现，放在 `error.c` 中，在 `main()` USER CODE BEGIN 2 最早调用：

```c
typedef enum
{
    ResetSrcUnknown = 0,
    ResetSrcPowerOn,
    ResetSrcExternal,
    ResetSrcSoftware,
    ResetSrcIwdg,
    ResetSrcWwdg,
    ResetSrcLowPower
} ResetSource;

static ResetSource g_last_reset_src = ResetSrcUnknown;

ResetSource error_check_reset_source(void)
{
    ResetSource src = ResetSrcUnknown;

    if (__HAL_RCC_GET_FLAG(RCC_FLAG_IWDGRST) != RESET)
    {
        src = ResetSrcIwdg;
        error_report(ERR_WATCHDOG);
    }
    else if (__HAL_RCC_GET_FLAG(RCC_FLAG_WWDGRST) != RESET)
    {
        src = ResetSrcWwdg;
        error_report(ERR_WATCHDOG);
    }
    else if (__HAL_RCC_GET_FLAG(RCC_FLAG_SFTRST) != RESET)
    {
        src = ResetSrcSoftware;
    }
    else if (__HAL_RCC_GET_FLAG(RCC_FLAG_PINRST) != RESET)
    {
        src = ResetSrcExternal;
    }
    else if (__HAL_RCC_GET_FLAG(RCC_FLAG_PORRST) != RESET)
    {
        src = ResetSrcPowerOn;
    }
    else if (__HAL_RCC_GET_FLAG(RCC_FLAG_LPWRRST) != RESET)
    {
        src = ResetSrcLowPower;
    }

    __HAL_RCC_CLEAR_RESET_FLAGS();
    g_last_reset_src = src;
    return src;
}

ResetSource error_get_reset_source(void)
{
    return g_last_reset_src;
}
```

上报与展示：

* 首次通信建立后，将 `g_last_reset_src` 作为状态字段发送给 ESP01S，例如 `{"cmd":"report","reset":"iwdg",...}`。
* OLED 启动画面可显示一次上次复位原因，2~3 秒后自动隐藏。
* 连续多次 IWDG 复位应进一步触发告警，提示硬件或任务卡死。

**注意**：必须在 `__HAL_RCC_CLEAR_RESET_FLAGS()` 之前把所有感兴趣的标志读完；清除后下次 `__HAL_RCC_GET_FLAG()` 永远返回 0。

`error_check_reset_source()` 需要在 `main()` 的 `USER CODE BEGIN 2` 最早调用。若函数内部调用 `error_report()`，错误模块必须满足以下二选一条件：

* 错误模块只依赖静态零初始化数据，不需要运行时 `error_init()`。
* 或者先调用 `error_init()`，再调用 `error_check_reset_source()`。

## 8. 与断言的关系

`APP_ASSERT()` 只用于 Debug 阶段捕获内部编程错误。传感器离线、通信超时、校验失败、用户输入越界等运行时问题必须使用 `ErrCode` 处理。
