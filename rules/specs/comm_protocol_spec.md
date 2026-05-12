# 通信协议规范

> 触发条件：任何涉及 STM32 与 ESP01S/Arduino 通信、UART 接收发送、云端上报、命令下发、JSON 解析、ACK/NACK 或重传机制的任务，必须先阅读本文件。
> 本文件定义 STM32 与 ESP01S/Arduino 固件之间的默认 UART JSON Lines 协议。

## 1. 基本约束

* 串口输入必须使用中断方式接收。
* 上报必须限速，默认 3 秒 1 次，宏名固定为 `ESP_REPORT_INTERVAL_MS`。
* 协议默认使用 JSON Lines：一行一个 JSON 对象，以 `\n` 作为帧结束符。
* 单帧最大长度必须用宏限制，默认 512 字节。
* 具体项目必须按真实最大上报字段估算 `COMM_LINE_MAX_LEN`，并至少保留 25% 余量。
* STM32 侧拼接 JSON 必须使用固定静态缓冲区和 `snprintf()`，禁止 `sprintf()` 和动态内存。
* STM32 侧 `snprintf()` 禁止使用 `%f` / `%.1f` 等浮点格式，避免 newlib-nano 未启用浮点 printf 时输出异常。
* STM32 侧默认手写固定字段轻量 JSON 解析器，不引入完整 JSON 库。
* ESP01S/Arduino 负责把 JSON 转发到云端或把云端命令转成同格式 JSON 下发。

```c
#define ESP_REPORT_INTERVAL_MS 3000U
#define COMM_LINE_MAX_LEN      512U
#define COMM_KEY_MAX_LEN       22U
#define COMM_RX_TIMEOUT_MS     100U
#define COMM_ACK_TIMEOUT_MS    500U
#define COMM_RETRY_MAX         3U
```

## 2. 帧格式

每帧格式：

```text
{json_object}\n
```

示例：

```text
{"cmd":"report","seq":1,"temp":25.6,"humi":60.3}\n
```

字段规则：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `cmd` | string | 是 | 命令类型 |
| `seq` | number | 建议 | 帧序号，用于 ACK/NACK 匹配和调试 |
| `err` | string | NACK 必填 | 错误码字符串 |
| 业务字段 | number/string/bool | 按需 | 传感器数据、阈值、开关状态 |

JSON 字段名使用小写 snake_case。数值字段优先直接用工程单位，如摄氏度、百分比、ppm，不强制放大成整数。

STM32 发送带小数的值时，必须使用定点整数手动格式化。例如温度内部用 0.1 摄氏度为单位保存为 `temp_x10=256`，发送 JSON 时手动输出 `25.6`，不得依赖 `%f`。负值定点数必须先拆分符号和绝对值，确保 `-5` 正确输出为 `-0.5`。

## 3. 命令类型

`cmd` 使用固定字符串，不使用裸数字命令字。

| `cmd` | 方向 | 含义 |
|-------|------|------|
| `report` | STM32 -> ESP01S | 周期上报传感器和设备状态 |
| `control` | ESP01S -> STM32 | 下发设备控制命令 |
| `set_param` | ESP01S -> STM32 | 下发阈值或参数 |
| `ack` | 双向 | 确认某条命令已成功处理 |
| `nack` | 双向 | 拒绝某条命令并返回错误原因 |
| `heartbeat` | 双向可选 | 链路保活 |

ACK/NACK 格式：

```text
{"cmd":"ack","seq":12,"ok":true}\n
{"cmd":"nack","seq":12,"ok":false,"err":"ERR_PARAM"}\n
```

`err` 字符串必须对应 `error_handling_spec.md` 中的错误码名称。

## 4. STM32 上报示例

温湿度上报：

```text
{"cmd":"report","seq":1,"temp":25.6,"humi":60.3}\n
```

多传感器上报：

```text
{"cmd":"report","seq":2,"temp":25.6,"humi":60.3,"co2":420,"tvoc":18,"light":735,"fan":0,"spray":1}\n
```

要求：

* 上报字段必须与实际项目传感器和执行机构一致，不编造云端字段。
* 上报周期默认 3 秒 1 次，不因按键或 OLED 刷新频率变化而加速。
* 无效传感器数据不要填假值，可用状态字段表示离线，如 `"dht11_ok":false`。
* 新项目实现前必须用最长字段名、最大数值长度和所有状态字段估算最大 JSON 行长度。
* `COMM_LINE_MAX_LEN` 必须大于最大 JSON 行长度的 1.25 倍。
* 如果完整上报超过 512 字节，优先拆成多类上报，如环境量、气体量、执行机构状态，不强行塞进单行。

长度估算示例：

```text
{"cmd":"report","seq":65535,"temp":25.6,"humi":60.3,"co2":420,"tvoc":18,"light":735,"fan":0,"spray":1,"dht11_ok":true}\n
```

这类帧约 117 字节，若再加入土壤湿度、浊度、液位、粉尘和距离，应按最终字段重新估算，而不是沿用小项目缓冲区。

## 5. ESP01S 下发示例

设备控制：

```text
{"cmd":"control","seq":12,"fan":1,"light":0,"spray":1}\n
```

参数设置：

```text
{"cmd":"set_param","seq":13,"temp_high":30.0,"humi_low":45.0}\n
```

要求：

* STM32 收到下发命令后必须校验字段范围。
* 未识别字段忽略或返回 `ERR_PARAM`，具体策略按项目风险确定。
* 执行成功回复 ACK，执行失败回复 NACK。

## 6. 重传机制

需要确认的命令必须等待 ACK/NACK。

* ACK 超时默认 `COMM_ACK_TIMEOUT_MS = 500U`。
* 重传次数默认 `COMM_RETRY_MAX = 3U`。
* 超过重传次数后放弃当前帧，记录 `ERR_COMM`。
* 重传等待必须使用 `HAL_GetTick()`，禁止无限等待。
* 周期上报如果只是遥测数据，可按项目需要选择不重传，避免阻塞新数据。

## 7. 接收状态机与同步恢复

串口接收必须使用中断方式，接收回调只写入环形缓冲区和标志位，不做完整 JSON 解析。完整解析放在 `esp_comm_process()` 或同级主循环任务中执行。

解析必须按行状态机处理，禁止假设一次收到完整 JSON。

```c
typedef enum
{
    CommRxWaitLine = 0,
    CommRxLineReady
} CommRxState;
```

同步恢复规则：

* 接收字节不是 `\n`：写入行缓冲区。
* 收到 `\n`：当前行结束，设置待解析标志。
* 行缓冲区超过 `COMM_LINE_MAX_LEN - 1`：丢弃当前行，记录 `ERR_OVER_RANGE`，重新等待下一行。
* 进入一行接收后超过 `COMM_RX_TIMEOUT_MS` 没收到 `\n`：丢弃当前行，记录 `ERR_TIMEOUT`。
* JSON 解析失败：丢弃当前行，记录 `ERR_PARAM` 或 `ERR_COMM`，必要时回复 NACK。
* 丢弃当前行时只清当前行缓冲区，不停止 UART 中断接收。

接收示例：

```c
void comm_rx_parse_byte(uint8_t byte)
{
    if (byte == '\r')
    {
        return;
    }

    if (comm_rx_index == 0U)
    {
        comm_rx_start_tick = HAL_GetTick();
    }

    if (byte == '\n')
    {
        comm_rx_line[comm_rx_index] = '\0';
        comm_rx_ready = 1U;
        comm_rx_index = 0U;
        return;
    }

    if (comm_rx_index >= (COMM_LINE_MAX_LEN - 1U))
    {
        comm_rx_index = 0U;
        error_report(ERR_OVER_RANGE);
        return;
    }

    comm_rx_line[comm_rx_index] = (char)byte;
    comm_rx_index++;
}
```

主循环中必须检查接收超时：

```c
void comm_rx_timeout_process(void)
{
    if ((comm_rx_index > 0U) && ((HAL_GetTick() - comm_rx_start_tick) >= COMM_RX_TIMEOUT_MS))
    {
        comm_rx_index = 0U;
        error_report(ERR_TIMEOUT);
    }
}
```

## 8. STM32 JSON 生成与发送规范

### 8.1 发送缓冲区所有权规则（强制）

`HAL_UART_Transmit_IT()` 和 DMA 发送在传输完成前会**持续读取原缓冲区**。一旦在 `TxCplt` 回调到来前复写该缓冲区，UART 会发出半旧半新的脏帧，云端解析失败且极难复现。

因此 STM32 侧 UART 发送必须遵守以下规则：

* 发送缓冲区必须是 `static`，不允许局部栈缓冲。
* 必须有发送状态机，由 `HAL_UART_TxCpltCallback()` **和** `HAL_UART_ErrorCallback()`、`HAL_UART_AbortCpltCallback()` 共同释放。任何一条非成功的发送结束路径都必须把状态机归位到 `CommTxIdle`，否则状态会在一次 UART 错误后永久卡在 `CommTxSending`，后续所有 `comm_tx_acquire()` 全部失败，上报链路静默死亡。
* UART 错误（noise/frame/parity/overrun）、发送中止、线缆断开以及 `HAL_UART_Transmit_IT()` 返回非 `HAL_OK` 的同步失败路径，必须记录 `ERR_COMM` 并执行状态机归位。
* 调用方必须先 acquire 再 snprintf 再 commit，acquire 成功后缓冲区立即进入 building 状态；生成失败必须 abort 归还；acquire 成功后所有 return 路径上必须配对 commit 或 abort，禁止 early return 不归还缓冲区。
* `comm_tx_acquire()`、`comm_tx_abort()`、`comm_tx_commit()` 内部修改发送状态时必须使用短临界区；ISR 中不直接拼接或发送 JSON，只置标志交给主循环处理。
* 周期上报丢一帧不影响系统功能，但绝不允许复写发送中缓冲区。
* 若项目需要并发发送（上报 + ACK + 告警），使用多个缓冲区轮换，单个缓冲区仍由独立状态保护。

```c
typedef enum
{
    CommTxIdle = 0,
    CommTxBuilding,
    CommTxSending
} CommTxState;

static char tx_buf[COMM_LINE_MAX_LEN];
static volatile CommTxState tx_state = CommTxIdle;

/* 通用释放路径：发送失败、UART 错误、Abort 完成时统一归位。
   记录 ERR_COMM，调用方无需再单独 error_report()。 */
static void comm_tx_release_from_fault(ErrCode code)
{
    __disable_irq();
    tx_state = CommTxIdle;
    __enable_irq();

    if (code != ERR_OK)
    {
        error_report(code);
    }
}

/* 获取当前可用发送缓冲区，返回 NULL 表示上一帧尚未发送完毕。
   调用方在获得缓冲区后必须 snprintf 成功，再调 comm_tx_commit()，
   否则必须调 comm_tx_abort() 归还缓冲区。 */
char *comm_tx_acquire(uint16_t *out_cap)
{
    char *buf = NULL;

    __disable_irq();
    if (tx_state != CommTxIdle)
    {
        __enable_irq();
        return NULL;
    }

    tx_state = CommTxBuilding;
    buf = tx_buf;
    __enable_irq();

    if (out_cap != NULL)
    {
        *out_cap = (uint16_t)COMM_LINE_MAX_LEN;
    }
    return buf;
}

void comm_tx_abort(void)
{
    __disable_irq();
    if (tx_state == CommTxBuilding)
    {
        tx_state = CommTxIdle;
    }
    __enable_irq();
}

ErrCode comm_tx_commit(uint16_t len)
{
    ErrCode ret = ERR_OK;

    __disable_irq();
    if (tx_state != CommTxBuilding)
    {
        __enable_irq();
        return ERR_COMM;
    }

    if ((len == 0U) || (len >= COMM_LINE_MAX_LEN))
    {
        tx_state = CommTxIdle;
        __enable_irq();
        return ERR_PARAM;
    }

    tx_state = CommTxSending;
    __enable_irq();

    if (HAL_OK != HAL_UART_Transmit_IT(&huart1, (uint8_t *)tx_buf, len))
    {
        comm_tx_release_from_fault(ERR_COMM);
        ret = ERR_COMM;
    }

    return ret;
}

void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart == &huart1)
    {
        /* 成功路径：不记 ERR_COMM，直接归位。uint8_t/枚举单字节写入原子，
           Error/Abort 回调同样直接赋值，无需临界区。 */
        tx_state = CommTxIdle;
    }
}

/* 噪声/帧错/奇偶/溢出等 UART 错误会在此触发。发送中发生错误会导致
   TxCplt 永不到达，这里必须归位发送状态机，否则整条发送链路会卡死。 */
void HAL_UART_ErrorCallback(UART_HandleTypeDef *huart)
{
    if (huart == &huart1)
    {
        if (tx_state == CommTxSending)
        {
            tx_state = CommTxIdle;
            error_report(ERR_COMM);
        }
        /* 接收错误由 UART 接收模块单独处理，不在此清接收状态 */
    }
}

/* 主动 HAL_UART_AbortTransmit_IT() 完成回调，用于线缆热插拔、
   长时间未完成发送时的手动中止。 */
void HAL_UART_AbortCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart == &huart1)
    {
        if (tx_state == CommTxSending)
        {
            tx_state = CommTxIdle;
            error_report(ERR_COMM);
        }
    }
}
```

### 8.2 JSON 生成规范

STM32 侧生成 JSON 必须使用固定缓冲区，并检查 `snprintf()` 返回值。禁止在 `snprintf()` 中使用 `%f`、`%.1f` 等浮点格式。

```c
uint16_t cap = 0U;
char *buf = NULL;
int16_t len = 0;
int16_t temp_x10 = 256;
int16_t humi_x10 = 603;
const char *temp_sign = "";
uint16_t temp_abs_x10 = 0U;
uint16_t humi_abs_x10 = 0U;

buf = comm_tx_acquire(&cap);
if (buf == NULL)
{
    /* 上一帧仍在发送，本次周期上报直接丢弃，等待下一次周期 */
    return ERR_COMM;
}

temp_sign = (temp_x10 < 0) ? "-" : "";
temp_abs_x10 = (uint16_t)((temp_x10 < 0) ? -temp_x10 : temp_x10);
humi_abs_x10 = (uint16_t)((humi_x10 < 0) ? -humi_x10 : humi_x10);

len = snprintf(buf, cap,
               "{\"cmd\":\"report\",\"seq\":%lu,\"temp\":%s%u.%u,\"humi\":%u.%u}\n",
               (unsigned long)seq,
               temp_sign,
               temp_abs_x10 / 10U,
               temp_abs_x10 % 10U,
               humi_abs_x10 / 10U,
               humi_abs_x10 % 10U);

if ((len <= 0) || (len >= (int16_t)cap))
{
    comm_tx_abort();
    error_report(ERR_OVER_RANGE);
    return ERR_OVER_RANGE;
}

return comm_tx_commit((uint16_t)len);
```

要求：

* 优先使用 `HAL_UART_Transmit_IT()` 或 DMA 发送。
* 发送必须走 `comm_tx_acquire()` / `comm_tx_commit()` / `comm_tx_abort()` 所有权协议，禁止直接调用 `HAL_UART_Transmit_IT()` 复写正在发送的缓冲区。
* 发送状态与发送缓冲区只能在主循环任务中构建，ISR 只允许置发送请求标志，不允许在 ISR 中调用 `snprintf()` 或直接提交 JSON 帧。
* 若确有并发发送场景（例如 ACK 与周期上报同时产生），扩展为 N 路缓冲区轮换，每路各有独立状态机。
* 如项目临时使用阻塞发送 `HAL_UART_Transmit()`，必须检查返回值，并确保不会破坏主循环实时性；阻塞发送无 buffer 生命周期问题，但不推荐在正常业务路径使用。
* 不允许使用 `malloc()`、`strcat()` 拼接未知长度字符串。
* 不允许使用 `%f` 输出浮点数；如确需小数，使用定点整数拆分整数和小数部分。
* 带符号定点数必须先拆分 `sign` 和绝对值，禁止直接用 `value / 10` 与 `abs(value % 10)` 拼接负数小数。

## 9. STM32 轻量 JSON 解析器规范

STM32 下发解析必须保持轻量。默认不使用完整 JSON 库，只实现固定字段轻量解析器。

解析器能力边界：

* 只支持一层 JSON 对象。
* 只支持 string、number、bool 三类字段。
* 不支持嵌套对象、数组、转义字符串、Unicode。
* 只识别白名单字段，未知字段默认忽略。
* `cmd` 缺失、关键字段缺失或字段范围非法时返回 NACK。
* ESP01S/Arduino 侧可使用 ArduinoJson，负责更复杂的云端 JSON 转换。

建议接口：

```c
int8_t json_get_string(const char *json, const char *key, char *out, uint16_t out_size);
int8_t json_get_int32(const char *json, const char *key, int32_t *out);
int8_t json_get_float(const char *json, const char *key, float *out);
int8_t json_get_bool(const char *json, const char *key, uint8_t *out);
```

`json_get_float()` 只用于接收端解析带小数的下发参数。解析成功后应立即转换为定点整数保存，例如 `temp_high:30.5` 转为 `temp_high_x10=305`，后续控制逻辑不依赖 float 长期存储。

返回值统一：

```text
 0  找到并解析成功
-1  字段不存在
-2  格式错误或输出缓冲区不足
```

字段查找规则：

* 查找完整键名模式：`"key"`，禁止只用裸字符串模糊匹配。
* 找到键名后必须继续匹配冒号 `:`。
* 冒号两侧允许空格、`\t`。
* string 值必须以双引号包裹。
* number 值允许负号、小数点，禁止指数格式。
* bool 值只允许 `true` 或 `false`。
* 字段名长度不得超过 `COMM_KEY_MAX_LEN`，默认 22 字符，避免键名匹配缓冲区截断。

## 10. 解析后校验与 NACK 规则

轻量解析器只负责按字段提取值，命令处理函数必须继续做必填字段、类型和范围校验。

统一处理规则：

| 场景 | 示例 | 处理 |
|------|------|------|
| `cmd` 字段不存在 | `{"seq":12,"fan":1}` | 返回 `ERR_PARAM`，回复 NACK |
| `cmd` 类型错误 | `{"cmd":123,"seq":12}` | 返回 `ERR_PARAM`，回复 NACK |
| 命令不支持 | `{"cmd":"unknown","seq":12}` | 返回 `ERR_PARAM`，回复 NACK |
| 数值字段类型错误 | `{"cmd":"set_param","temp":"abc"}` | 返回 `ERR_PARAM`，回复 NACK |
| 控制字段越界 | `{"cmd":"control","fan":999}` | 返回 `ERR_PARAM`，回复 NACK |
| 可选字段不存在 | `{"cmd":"control","fan":1}` 中没有 `spray` | 不报错，保持原状态 |
| 未知字段 | `{"cmd":"control","fan":1,"foo":2}` | 默认忽略 |

NACK 示例：

```text
{"cmd":"nack","seq":12,"ok":false,"err":"ERR_PARAM"}\n
```

必填字段校验模板：

```c
ErrCode comm_require_string(const char *line, const char *key, char *out, uint16_t out_size)
{
    int8_t ret = 0;

    ret = json_get_string(line, key, out, out_size);
    if (ret != 0)
    {
        return ERR_PARAM;
    }

    return ERR_OK;
}

ErrCode comm_optional_u8_range(const char *line, const char *key, uint8_t min_value, uint8_t max_value, uint8_t *out, uint8_t *present)
{
    int8_t ret = 0;
    int32_t value = 0;

    if ((out == NULL) || (present == NULL))
    {
        return ERR_PARAM;
    }

    *present = 0U;
    ret = json_get_int32(line, key, &value);
    if (ret == -1)
    {
        return ERR_OK;
    }
    if (ret != 0)
    {
        return ERR_PARAM;
    }
    if ((value < (int32_t)min_value) || (value > (int32_t)max_value))
    {
        return ERR_PARAM;
    }

    *out = (uint8_t)value;
    *present = 1U;
    return ERR_OK;
}
```

固定字段解析示例：

```c
typedef struct
{
    char cmd[16];
    uint32_t seq;
    uint8_t fan;
    uint8_t fan_present;
    uint8_t light;
    uint8_t light_present;
    uint8_t spray;
    uint8_t spray_present;
} CommControlMsg;

ErrCode comm_parse_control(const char *line, CommControlMsg *msg)
{
    ErrCode err = ERR_OK;
    int32_t temp_value = 0;

    if ((line == NULL) || (msg == NULL))
    {
        return ERR_PARAM;
    }

    memset(msg, 0, sizeof(CommControlMsg));

    err = comm_require_string(line, "cmd", msg->cmd, sizeof(msg->cmd));
    if (err != ERR_OK)
    {
        return err;
    }

    if (strcmp(msg->cmd, "control") != 0)
    {
        return ERR_PARAM;
    }

    if (json_get_int32(line, "seq", &temp_value) == 0)
    {
        if (temp_value < 0)
        {
            return ERR_PARAM;
        }
        msg->seq = (uint32_t)temp_value;
    }

    err = comm_optional_u8_range(line, "fan", 0U, 1U, &msg->fan, &msg->fan_present);
    if (err != ERR_OK)
    {
        return err;
    }

    err = comm_optional_u8_range(line, "light", 0U, 1U, &msg->light, &msg->light_present);
    if (err != ERR_OK)
    {
        return err;
    }

    err = comm_optional_u8_range(line, "spray", 0U, 1U, &msg->spray, &msg->spray_present);
    if (err != ERR_OK)
    {
        return err;
    }

    return ERR_OK;
}
```

轻量字符串字段提取示例：

```c
int8_t json_get_string(const char *json, const char *key, char *out, uint16_t out_size)
{
    const char *pos = NULL;
    const char *colon = NULL;
    const char *start = NULL;
    const char *end = NULL;
    char pattern[COMM_KEY_MAX_LEN + 3U];
    int16_t pattern_len = 0;
    uint16_t len = 0U;

    if ((json == NULL) || (key == NULL) || (out == NULL) || (out_size == 0U))
    {
        return -2;
    }

    if (strlen(key) > COMM_KEY_MAX_LEN)
    {
        return -2;
    }

    pattern_len = snprintf(pattern, sizeof(pattern), "\"%s\"", key);
    if ((pattern_len <= 0) || (pattern_len >= (int16_t)sizeof(pattern)))
    {
        return -2;
    }

    pos = strstr(json, pattern);
    if (pos == NULL)
    {
        return -1;
    }

    colon = strchr(pos + pattern_len, ':');
    if (colon == NULL)
    {
        return -2;
    }

    start = colon + 1;
    while ((*start == ' ') || (*start == '\t'))
    {
        start++;
    }

    if (*start != '"')
    {
        return -2;
    }
    start++;

    end = strchr(start, '"');
    if (end == NULL)
    {
        return -2;
    }

    len = (uint16_t)(end - start);
    if (len >= out_size)
    {
        return -2;
    }

    memcpy(out, start, len);
    out[len] = '\0';
    return 0;
}
```

实现要求：

* 解析器函数必须放在独立模块，如 `json.c/.h` 或通信模块内部私有函数。
* 示例代码只是边界模板，具体项目必须补齐 number/bool 提取和字段范围校验。
* 不允许为了 JSON 解析引入 `malloc()`。
* 不允许把未知长度字符串复制到固定缓冲区。
* 不允许解析业务无关字段，避免 STM32 侧协议复杂化。

## 11. 与 ESP01S/Arduino 的交互边界

STM32 负责本地传感器采集、执行机构控制、OLED UI 和 JSON 行协议收发。ESP01S/Arduino 负责 Wi-Fi、云平台连接和云端 JSON 转换。

* STM32 上报必须使用 `{"cmd":"report",...}\n`。
* ESP01S 下发控制必须使用 `{"cmd":"control",...}\n` 或 `{"cmd":"set_param",...}\n`。
* ESP01S 收到 STM32 上报后可直接转成 MQTT/HTTP JSON。
* 云端限流时，ESP01S 可以丢弃过密上报，但不得反向阻塞 STM32 主循环。
* 串口监视器调试时，每一行都应是可读 JSON，便于人工排查。