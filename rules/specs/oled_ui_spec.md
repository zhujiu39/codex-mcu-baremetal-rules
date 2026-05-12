# OLED UI 设计规范（0.96 寸 128x64 固定风格）

> 触发条件：任何涉及 OLED 显示、UI 页面、菜单、阈值设置、设备控制、显示刷新、页面切换或长文本显示的任务，必须先阅读本文件。
> 以下为 OLED 多页面 UI 的强制设计规范，所有涉及 OLED 显示的项目必须遵守。

## 1. 文件架构

* UI 逻辑必须独立放在 `Core/Src/ui.c` 和 `Core/Inc/ui.h` 中。
* `main.c` 中只允许调用 `ui_init()` 和 `ui_process()`。
* UI 模块通过 `#include` 引用传感器/执行机构头文件，读取数据并显示。
* UI 模块提供 `ui_get_xxx()` 系列函数，供控制逻辑读取用户设定值。
* UI 模块提供 `ui_set_xxx()` 系列函数，供云端通信模块远程修改参数。

## 2. 页面划分规则

* 每个传感器必须有一个独立监测页面。
* 阈值/参数设置必须有一个独立设置页面。
* 设备控制必须有一个独立控制页面。
* 特殊功能必须分配独立页面。
* 页面索引用宏定义：`UI_PAGE_XXX`，从 0 开始。
* 页面总数用 `UI_PAGE_COUNT`。
* 每个页面对应一个 `static void ui_page_xxx(void)` 私有函数。

## 3. 标题栏设计

* 位置：顶部 0~10 像素高度。
* 背景：反色填充，白底黑字。
* 布局：`[标题文字]  [模式标识]  [页码]`。
* 标题文字：左对齐，x=2，使用 6x8 字体，英文大写。
* 模式标识：靠右，显示 `A` 或 `M`，无模式项目可省略。
* 页码：最右侧，格式 `N/M`。
* 统一绘制函数：`static void ui_draw_header(const char *title, uint8_t page_num)`。

## 4. 内容区域布局

* 可用区域：y=12 到 y=63。
* 数据展示页：核心数值用 8x16 字体，附带单位；辅助信息用 6x8 字体。
* 设置/控制页：每行一个条目，左侧用 `>` 标记当前选中项。
* 底部保留一行显示操作提示，如 `SEL:Item +/-:Adj`。
* ON/异常状态可使用反色矩形突出显示，OFF 状态普通显示。

## 5. 内容溢出处理

128x64 OLED 空间有限，任何文本和数字不得互相覆盖。

* 数字超宽：优先缩小字体，其次截断并保留单位。
* 列表超长：使用上下滚动，只显示当前窗口内条目。
* 长文本：优先缩写为短状态词，其次截断并追加 `..`。
* 跑马灯默认关闭，仅当 `UI_USE_MARQUEE` 为 1 时允许对长文本横向滚动。
* 跑马灯只用于设备名称、长告警文字等确实需要完整展示的内容，不用于实时数值。

```c
#define UI_USE_MARQUEE 0U
```

## 6. 页面切换过渡

页面切换默认硬切换。简单过渡效果为可选能力，由 `UI_USE_TRANSITION` 控制，默认关闭。

```c
#define UI_USE_TRANSITION 0U
```

启用过渡时必须满足：

* 不影响 `UI_REFRESH_MS` 刷新节奏。
* 不在绘制过程中直接刷硬件屏幕。
* 不阻塞按键处理和通信处理。

## 7. 按键交互规范

按键引脚、消抖、事件宏等底层规则详见 `key_spec.md`。

* UI 模块不直接读取 GPIO，只通过 `key_read()` / `key_long_press()` 获取事件。
* PAGE 切页时选中条目归零。
* SELECT 在设置/控制页面中循环切换选中条目。
* UP/DOWN 调整参数或控制开关。
* 长按切换模式后，底部反色显示 2 秒提示，如 `>> AUTO MODE <<`。
* 每个设置/控制页面的条目数用宏定义：`UI_XXX_ITEM_COUNT`。

## 8. 刷新策略（强制）

* 所有绘制操作写入显存缓冲区，最后调用一次 `oled_refresh()` 推送到屏幕。
* 定时刷新使用 `HAL_GetTick()`，默认 300ms，宏 `UI_REFRESH_MS`。
* 按键触发即时刷新：按键事件发生时设置 `ui_need_refresh = 1U`。
* 刷新流程：`oled_clear()` -> 绘制页面内容 -> `oled_refresh()`。
* 严禁在绘制过程中直接调用硬件刷屏函数。
* 严禁使用 `HAL_Delay()` 控制刷新节奏。

```c
#define UI_REFRESH_MS 300U
```

## 9. 变量与数据结构规范

* 页面索引：`static uint8_t ui_current_page`。
* 选中条目：`static uint8_t ui_selected_item`。
* 刷新控制：`static uint32_t ui_last_tick` + `static uint8_t ui_need_refresh`。
* 阈值参数：用 `typedef struct { ... } UiThreshold` 封装。阈值内部**必须使用定点整数**保存，不使用 `float`，与 `comm_protocol_spec.md` 第 8 节"STM32 侧禁用 `%f`、float 下发后立即转定点"保持一致。显示阶段局部换算成字符串，不在结构体里长期保存浮点。
* 运行模式：用 `typedef enum { UiModeAuto = 0, UiModeManual } UiRunMode` 定义。

推荐的阈值结构（以温湿度为例，保留 1 位小数 → `_x10` 形式）：

```c
typedef struct
{
    int16_t  temp_high_x10;  /* 温度上限，单位 0.1 摄氏度 */
    int16_t  temp_low_x10;   /* 温度下限，单位 0.1 摄氏度 */
    uint16_t humi_high_x10;  /* 湿度上限，单位 0.1 % */
    uint16_t humi_low_x10;   /* 湿度下限，单位 0.1 % */
} UiThreshold;
```

新增其他传感器阈值时沿用同样规则：小数位统一用 `_x10` / `_x100`，ppm / lux 等天然整数直接用 `uint16_t` / `uint32_t`。

## 10. 公共接口规范

```c
void              ui_init(void);
void              ui_process(void);
const UiThreshold *ui_get_threshold(void);
UiRunMode         ui_get_mode(void);
void              ui_set_mode(UiRunMode mode);

/* 设置接口全部使用定点整数，不使用 float。
   想不修改某个字段时传入 INT16_MIN / UINT16_MAX 等特殊哨兵值，
   内部做范围钳位与"不修改"判断。哨兵值由各项目在 ui.h 中宏定义，
   例如 UI_TH_KEEP_I16 / UI_TH_KEEP_U16。 */
void ui_set_threshold(int16_t  temp_high_x10,
                      int16_t  temp_low_x10,
                      uint16_t humi_high_x10,
                      uint16_t humi_low_x10);
```

要求：

* 云端下发的 JSON 解析到 `float` 后，必须立即乘 10 取整转成 `int16_t` / `uint16_t` 再调 `ui_set_threshold()`。
* UI 绘制页面时使用 `snprintf()` 和 sign/abs 拆分输出，禁止 `%f`，与 `comm_protocol_spec.md` 的负数拆分规则一致。
* 内部阈值保存**禁止**使用 `float`，避免 newlib-nano 未启用浮点 printf 时显示异常，也避免和协议层反复换算引入精度漂移。

定点显示示例：

```c
int16_t val_x10 = -5;
const char *sign = (val_x10 < 0) ? "-" : "";
uint16_t abs_x10 = (uint16_t)((val_x10 < 0) ? -val_x10 : val_x10);

snprintf(buf, buf_size, "%s%u.%u", sign, abs_x10 / 10U, abs_x10 % 10U);
```

## 11. ui_process() 标准流程

```
ui_process() {
    1. key_read()
    2. key_long_press()
    3. ui_handle_key(evt)
    4. 检查临时提示超时
    5. 判断 need_refresh 或定时到期
    6. oled_clear()
    7. switch 绘制当前页面
    8. 叠加临时提示
    9. oled_refresh()
}
```
