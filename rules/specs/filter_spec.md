# 滤波算法对照表

> 触发条件：任何涉及传感器采集、ADC 数据处理、测距、环境数据平滑、阈值判断或滤波算法新增修改的任务，必须先阅读本文件。
> 此文件为按需加载的详细参考，核心规则见 `AGENTS.md`。
> 规则：所有传感器数据采集必须配套滤波算法。下表参数是默认起点，不是跨项目永久常量；采样周期、噪声环境或控制响应要求变化时必须复核。

## 传感器滤波参数表

表中采样周期是默认适用条件。若实际采样周期偏离 2 倍以上，必须重新评估 α 或窗口大小。

| 传感器 | 滤波算法 | 默认参数 | 默认采样周期 | 适用噪声/信号条件 | 参数来源 |
|--------|----------|----------|--------------|-------------------|----------|
| DHT11（温湿度） | EMA 滤波 | α=0.30 | 1000~2000ms | 温湿度慢变化，读数偶发抖动，不适合快速闭环控制 | 常见项目实测起点 |
| BME280（温湿度/气压） | EMA 滤波 | α=0.20 | 500~1000ms | 环境量慢变化，噪声较小，允许 2~5s 平滑响应 | 经验起点 |
| SGP30（eCO2/TVOC） | 中值滤波 + EMA 滤波 | 窗口=5，α=0.15 | 1000ms | 气体读数有尖峰和缓慢漂移，优先抑制瞬时跳变 | 常见项目实测起点 |
| MQ-2（烟雾） | 中值滤波 + EMA 滤波 | 窗口=5，α=0.15 | 200~1000ms | ADC 噪声明显，继电器/电源干扰可能造成尖峰 | 常见项目实测起点 |
| MQ-135（空气质量） | 中值滤波 + EMA 滤波 | 窗口=5，α=0.15 | 200~1000ms | ADC 噪声明显，气体浓度变化较慢 | 常见项目实测起点 |
| GL5506 光敏电阻（ADC 光照） | EMA 滤波 + 迟滞阈值 | α=0.10，开=800，关=900 | 100~500ms | 光照可能受交流灯和阴影扰动，重点避免阈值附近抖动 | 常见项目实测起点 |
| GP2Y1014AU（粉尘） | 中值滤波 + EMA 滤波 | 窗口=5，α=0.12 | 1000ms | 粉尘采样脉冲噪声明显，优先抑制异常高值 | 经验起点 |
| 土壤湿度（ADC） | EMA 滤波 | α=0.20 | 500~2000ms | 慢变化模拟量，噪声通常来自 ADC 和供电 | 经验起点 |
| 浊度传感器（ADC） | 中值滤波 + EMA 滤波 | 窗口=5，α=0.18 | 200~1000ms | 采样尖峰明显，液体状态变化中等速度 | 常见项目实测起点 |
| 液位传感器（ADC） | EMA 滤波 | α=0.25 | 200~1000ms | 液位变化较慢，但需要比温湿度更快响应 | 经验起点 |
| HC-SR04 超声波测距 | 中值滤波 + 滑动平均 | 窗口=5，平均窗口=3 | 100~300ms | 回波存在偶发异常值，目标移动速度低到中等 | 经验起点 |
| 红外对射（粮食检测） | 计数去抖 | 连续触发 >= 3 次判定有效 | 20~100ms | 数字脉冲可能短暂误触发，目标通过速度不能过快 | 经验起点 |
| PIR 人体红外（数字） | 计数去抖 + 保持时间 | 连续高电平 >= 3 次，保持 2s | 100~500ms | 人体检测慢事件，输出常带保持时间 | 经验起点 |
| 电压/电流采样（ADC） | 中值滤波 + EMA 滤波 | 窗口=5，α=0.10 | 50~500ms | 电源噪声和负载切换尖峰明显，优先稳定显示 | 经验起点 |

按键不在本表定义去抖参数。按键交互必须遵守 `key_spec.md`，以 EXTI 中断 + `HAL_GetTick()` 消抖为准。

## 参数复核规则

以下情况必须重新调参，不允许直接套用表中默认值：

* 采样周期偏离表中范围 2 倍以上。
* 传感器用途从“显示监测”变成“闭环控制”。
* 负载、电机、继电器、泵、风扇等引入新的电磁干扰。
* 阈值附近出现频繁误触发或响应明显过慢。
* 传感器安装位置变化导致信号波动特性变化。

调参时至少记录 30~60 秒原始值和滤波值，观察：

* 静态噪声幅度：设备处于稳定环境时 raw 的最大最小波动。
* 阶跃响应时间：真实变化发生后 filtered 到达目标变化 90% 所需时间。
* 误触发次数：阈值附近是否因噪声反复开关。

调参建议：

* 响应太慢：增大 α，或启用自适应 EMA。
* 显示抖动：减小 α，或增加中值滤波。
* 偶发尖峰：先加中值滤波，再调 EMA。
* 阈值附近反复动作：优先加迟滞阈值，不只靠减小 α。

## EMA 滤波公式

```
filtered = alpha * raw + (1 - alpha) * filtered_prev
```

α 越大，响应越快但噪声抑制越弱；α 越小，越平滑但延迟越大。

## EMA 标准实现模板

```c
typedef struct
{
    float value;
    uint8_t initialized;
} EmaFilter;

float ema_filter_update(EmaFilter *filter, float raw, float alpha)
{
    if (filter == NULL)
    {
        return raw;
    }

    if (filter->initialized == 0U)
    {
        filter->value = raw;
        filter->initialized = 1U;
    }
    else
    {
        filter->value = alpha * raw + (1.0f - alpha) * filter->value;
    }

    return filter->value;
}
```

## 自适应 EMA 可选策略

阶跃变化明显的传感器可启用自适应 EMA。默认不强制启用，启用时必须给出阈值来源。

```c
float ema_filter_update_adaptive(EmaFilter *filter, float raw, float normal_alpha, float fast_alpha, float step_threshold)
{
    float diff = 0.0f;
    float alpha = normal_alpha;

    if (filter == NULL)
    {
        return raw;
    }

    diff = raw - filter->value;
    if (diff < 0.0f)
    {
        diff = -diff;
    }

    if ((filter->initialized != 0U) && (diff > step_threshold))
    {
        alpha = fast_alpha;
    }

    return ema_filter_update(filter, raw, alpha);
}
```

## 中值滤波标准实现模板

窗口默认使用 5。排序使用局部静态长度数组，不使用动态内存。

```c
#define MEDIAN_WINDOW_SIZE 5U

uint16_t median5_filter(const uint16_t input[MEDIAN_WINDOW_SIZE])
{
    uint16_t buf[MEDIAN_WINDOW_SIZE];
    uint8_t i = 0U;
    uint8_t j = 0U;

    if (input == NULL)
    {
        return 0U;
    }

    for (i = 0U; i < MEDIAN_WINDOW_SIZE; i++)
    {
        buf[i] = input[i];
    }

    for (i = 0U; i < (MEDIAN_WINDOW_SIZE - 1U); i++)
    {
        for (j = 0U; j < (MEDIAN_WINDOW_SIZE - 1U - i); j++)
        {
            if (buf[j] > buf[j + 1U])
            {
                uint16_t temp = buf[j];
                buf[j] = buf[j + 1U];
                buf[j + 1U] = temp;
            }
        }
    }

    return buf[MEDIAN_WINDOW_SIZE / 2U];
}
```

## HC-SR04 温度补偿

当项目中已有温度传感器时，HC-SR04 测距必须使用温度补偿声速。

```
v = 331.3 + 0.606 * T
distance_cm = echo_time_us * v / 20000
```

其中 `T` 为摄氏温度，`v` 单位为 m/s。除以 20000 是因为回波时间包含往返距离，并将 m/us 换算为 cm。

无温度传感器时，默认按 20 摄氏度估算声速，并在注释中说明。

## ADC 多通道规则

只要涉及 ADC 多通道，直接使用 DMA，无需询问但需通知用户。DMA 采集后必须配套本文件中对应的滤波算法。
