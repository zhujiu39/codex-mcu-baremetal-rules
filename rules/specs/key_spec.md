# 按键规范（4 键标准布局）

> 触发条件：任何涉及按键引脚、按键中断、消抖、短按/长按事件、UI 交互、蜂鸣器按键反馈的任务，必须先阅读本文件。
> 所有涉及按键交互的项目必须遵守此规范，不可凭记忆猜测。

## 1. 引脚定义（固定，不可更改）

| 按键 | 引脚 | 宏定义 |
|------|------|--------|
| PAGE | PB12 | `KEY_PAGE_PIN` / `KEY_PAGE_PORT` |
| SELECT | PB13 | `KEY_SELECT_PIN` / `KEY_SELECT_PORT` |
| UP(+) | PB14 | `KEY_UP_PIN` / `KEY_UP_PORT` |
| DOWN(-) | PB15 | `KEY_DOWN_PIN` / `KEY_DOWN_PORT` |

宏定义示例放在 `key.h` 中：

```c
#define KEY_PAGE_PORT     GPIOB
#define KEY_PAGE_PIN      GPIO_PIN_12
#define KEY_SELECT_PORT   GPIOB
#define KEY_SELECT_PIN    GPIO_PIN_13
#define KEY_UP_PORT       GPIOB
#define KEY_UP_PIN        GPIO_PIN_14
#define KEY_DOWN_PORT     GPIOB
#define KEY_DOWN_PIN      GPIO_PIN_15
```

## 2. 硬件要求

* CubeMX 配置：PB12~PB15 为 GPIO_EXTI + 内部上拉（Pull-up）。
* 按键接线：按下接地，低电平有效。
* EXTI 触发方式：
  - 仅需短按的项目，可配置为**下降沿触发**（Falling Edge），配合原"按下即事件"的简化实现。
  - 需要长按的项目，**必须配置为上升+下降双边沿触发**（Rising/Falling Edge），ISR 按释放时的按住时长分类短按/长按，详见第 4.2 节。
* 默认采用双边沿方案，保证 UI 模式切换、阈值锁定等交互需求可用。
* PB12~PB15 在部分 STM32 上与 SPI2（SCK/MISO/MOSI/NSS）复用，若项目同时使用 SPI2（如 SD 卡、W5500、某些 OLED），必须与张三协商改用其他 GPIO。
* 使用 HAL 标准 `HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)` 时，回调参数只有 pin 没有 port，因此四个按键必须使用同一个 GPIO port，且 pin 编号必须互不相同。若必须跨 port 且存在相同 pin 编号，不能使用本文件的简化 `key_index_from_pin()`，必须改为项目专用 EXTI 分发实现。
* 不确定外部电路时必须询问用户。

## 3. 功能角色

| 按键 | 功能角色 | 行为 |
|------|----------|------|
| PAGE | 翻页 | 循环切换页面，切页时选中条目归零 |
| SELECT | 选择 | 在设置/控制页面中循环切换选中条目 |
| UP(+) | 加/开启/启动 | 增大阈值、开启设备、启动功能 |
| DOWN(-) | 减/关闭/停止 | 减小阈值、关闭设备、停止功能 |

## 4. 消抖规范（强制：中断消抖）

* 禁止使用轮询扫描消抖。
* 必须使用 EXTI 中断 + `HAL_GetTick()` 软件消抖。
* 消抖时间宏：`KEY_DEBOUNCE_MS`，默认 50ms。
* 中断回调中只记录时间戳和按下/释放两个原子事件，不做任何业务逻辑。
* 业务逻辑在主循环的 `key_read()` 中判断消抖后处理。
* 时间差判断必须使用 `uint32_t` 无符号减法：`(now - last_tick) >= timeout_ms`。
* 禁止把 `HAL_GetTick()`、时间戳或差值转成 `int32_t` / `int` 后比较。
* `timeout_ms` 必须远小于 `0x80000000UL`，按键消抖和长按阈值均满足该条件。
* 该写法在 `HAL_GetTick()` 约 49.7 天回绕时仍然有效。

### 4.1 跨 IRQ 共享状态的强制约束

本规范默认 PB12~PB15 共用 `EXTI15_10_IRQHandler`，四键互不抢占，`g_key_state[]` 不会出现多个按键 ISR 并发写。

如项目把按键拆到**不同 EXTI 线**（例如 PA0 走 EXTI0、PB1 走 EXTI1），则：

* `g_key_state[].press_tick`、`pressed`、`short_event`、`long_event` 等复合状态的读改写不再原子。
* 所有跨 IRQ 共享的复合状态更新必须用 `__disable_irq()` / `__enable_irq()` 临界区包裹，或通过每键独立的单一 `volatile uint8_t` 事件标志间接同步。
* 禁止在不同优先级的 ISR 之间直接共享同一个复合状态结构而不加保护。

### 4.2 双边沿捕获（用于长按判定）

若项目需要长按检测，EXTI 必须配置为**双边沿触发**（Rising + Falling），同时使能内部上拉。只有双边沿才能在 ISR 里拿到准确的"按下时刻"和"释放时刻"。纯下降沿配置无法区分长按与"用户按着不放又抖动"。

* 按下事件：ISR 读到引脚为低电平时记录 `press_tick`，置 `pressed=1`。
* 释放事件：ISR 读到引脚为高电平时，计算 `held = now - press_tick`：
  - `held < KEY_DEBOUNCE_MS`：视为抖动，丢弃。
  - `KEY_DEBOUNCE_MS <= held < KEY_LONG_PRESS_MS`：置短按事件。
  - `held >= KEY_LONG_PRESS_MS`：置长按事件。
* 若项目不需要长按，可保持原"纯下降沿 + 按下即记事件"的简化实现，但禁止再调用 `key_long_press()`。
* 本示例只实现**释放后长按**一种语义。若项目需要"按住达到阈值立即触发"，必须单独实现即时长按版本，禁止把释放后长按和即时长按混在同一套示例里。

```c
#define KEY_DEBOUNCE_MS    50U
#define KEY_LONG_PRESS_MS  2000U

typedef struct
{
    volatile uint32_t press_tick;    /* 按下瞬间 HAL_GetTick() */
    volatile uint8_t  pressed;       /* 1=当前处于按下状态 */
    volatile uint8_t  short_event;   /* 释放后判定的短按事件 */
    volatile uint8_t  long_event;    /* 释放后判定的长按事件 */
} KeyState;

static KeyState g_key_state[4] = {0};

static uint8_t key_index_from_pin(uint16_t pin)
{
    if (pin == KEY_PAGE_PIN)   { return 0U; }
    if (pin == KEY_SELECT_PIN) { return 1U; }
    if (pin == KEY_UP_PIN)     { return 2U; }
    if (pin == KEY_DOWN_PIN)   { return 3U; }
    return 0xFFU;
}

static GPIO_TypeDef *key_port_from_index(uint8_t idx)
{
    switch (idx)
    {
        case 0U: return KEY_PAGE_PORT;
        case 1U: return KEY_SELECT_PORT;
        case 2U: return KEY_UP_PORT;
        case 3U: return KEY_DOWN_PORT;
        default: return NULL;
    }
}

static uint16_t key_pin_from_index(uint8_t idx)
{
    switch (idx)
    {
        case 0U: return KEY_PAGE_PIN;
        case 1U: return KEY_SELECT_PIN;
        case 2U: return KEY_UP_PIN;
        case 3U: return KEY_DOWN_PIN;
        default: return 0U;
    }
}

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    uint8_t idx = key_index_from_pin(GPIO_Pin);
    if (idx >= 4U)
    {
        return;
    }

    GPIO_TypeDef *port = key_port_from_index(idx);
    uint16_t pin = key_pin_from_index(idx);
    KeyState *k = &g_key_state[idx];
    uint32_t now = HAL_GetTick();

    /* 低电平=按下；上拉+按下接地的硬件假设见第 2 节 */
    if (HAL_GPIO_ReadPin(port, pin) == GPIO_PIN_RESET)
    {
        /* 按下边沿：抖动窗口内重复进入忽略 */
        if ((k->pressed == 0U) && ((now - k->press_tick) >= KEY_DEBOUNCE_MS))
        {
            k->press_tick = now;
            k->pressed = 1U;
        }
    }
    else
    {
        /* 释放边沿：根据按下时长分类 */
        if (k->pressed != 0U)
        {
            uint32_t held = now - k->press_tick;

            if (held < KEY_DEBOUNCE_MS)
            {
                /* 释放抖动：保持 pressed=1，等待后续真实释放边沿 */
            }
            else if (held >= KEY_LONG_PRESS_MS)
            {
                k->pressed = 0U;
                k->long_event = 1U;
            }
            else
            {
                k->pressed = 0U;
                k->short_event = 1U;
            }
        }
    }
}
```

## 5. 读清模式

`key_read()` 读取对应事件标志后必须立即清零。短按事件只在按键**释放**时产生，禁止在按下瞬间就生成短按事件，否则会与长按事件重复触发。

```c
uint8_t key_read(void)
{
    uint8_t evt = KEY_NONE;

    /* 按优先级：PAGE > SELECT > UP > DOWN。
       读-清事件标志必须保持原子，避免 ISR 在主循环判断后、清零前写入事件。 */
    __disable_irq();
    if (g_key_state[0].short_event != 0U)
    {
        g_key_state[0].short_event = 0U;
        evt = KEY_EVT_PAGE;
    }
    else if (g_key_state[1].short_event != 0U)
    {
        g_key_state[1].short_event = 0U;
        evt = KEY_EVT_SELECT;
    }
    else if (g_key_state[2].short_event != 0U)
    {
        g_key_state[2].short_event = 0U;
        evt = KEY_EVT_UP;
    }
    else if (g_key_state[3].short_event != 0U)
    {
        g_key_state[3].short_event = 0U;
        evt = KEY_EVT_DOWN;
    }
    __enable_irq();

    return evt;
}
```

## 6. 长按检测（可选）

如项目需要模式切换，使用 SELECT 键长按 2 秒实现。

* 长按判定时间宏：`KEY_LONG_PRESS_MS`，默认 2000ms。
* 本规范默认使用**释放后长按**：按键释放时按住时长 >= `KEY_LONG_PRESS_MS` 才触发，由 ISR 在释放边沿写入 `long_event`（见 4.2 节）。
* 若项目要做**即时长按**，必须单独写一套即时长按实现，并删除释放后长按路径，禁止两种语义共用同一个 `key_long_press()`。
* 禁止死循环等待长按释放。
* 禁止同时产生短按与长按事件。

```c
/* 释放后长按版本：只读取 ISR 在释放边沿生成的 long_event */
uint8_t key_long_press(void)
{
    KeyState *k = &g_key_state[1];
    uint8_t evt = KEY_NONE;

    __disable_irq();
    if (k->long_event != 0U)
    {
        k->long_event = 0U;
        evt = KEY_LONG_EVT_SELECT;
    }
    __enable_irq();

    return evt;
}
```

说明：

* 释放后长按适合"按住并校准时间再触发"的严肃操作。
* 即时长按适合模式切换、清零等需要立即反馈的操作，但必须另写独立示例，不得和本释放后长按示例混用。

## 7. 可选蜂鸣器反馈

如项目有蜂鸣器，可在有效按键事件确认后给短促提示音。

* 蜂鸣器反馈默认可选，不强制。
* 只能在 `key_read()` 返回有效事件后，由 UI 或应用层触发。
* 禁止在 EXTI 回调中直接控制蜂鸣器。
* 提示音建议 30~50ms，并使用非阻塞定时关闭。
* 无效按键、越界调整、被锁定的操作不响，避免误导用户。

## 8. 文件架构

* 按键驱动必须独立放在 `Core/Src/key.c` 和 `Core/Inc/key.h` 中。
* `ui.c` 通过 `#include "key.h"` 调用按键接口。
* `main.c` 中不直接操作按键 GPIO。

## 9. 公共接口

```c
void key_init(void);
uint8_t key_read(void);
uint8_t key_long_press(void);
```

短按事件宏：

```c
#define KEY_NONE            0U
#define KEY_EVT_PAGE        1U
#define KEY_EVT_SELECT      2U
#define KEY_EVT_UP          3U
#define KEY_EVT_DOWN        4U
```

长按事件宏：

```c
#define KEY_LONG_EVT_SELECT 10U
```

## 10. 与 UI 模块协作

* `ui_process()` 中调用 `key_read()` 获取短按事件。
* `ui_process()` 中按需调用 `key_long_press()` 获取长按事件。
* 按键事件触发后设置 `ui_need_refresh = 1U`，实现即时刷新。
* UI 模块不直接读取 GPIO，只通过 `key.h` 接口获取事件。
