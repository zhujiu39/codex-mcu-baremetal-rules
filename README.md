# Codex STM32 Bare-Metal Rules

这是一套面向 Codex 的 STM32 嵌入式裸机开发规则包，适用于 `STM32CubeMX + Keil + HAL` 的裸机工程开发场景。

本项目不是一个完整的 STM32 固件工程，而是一组用于约束 AI Agent 开发行为的规则模板。它的目标是让 Codex 在参与 STM32 项目开发时，能够遵守 CubeMX 自动生成代码边界，按模块化方式编写高层逻辑、驱动代码、传感器算法、OLED UI、按键交互和串口通信协议。

## 适用场景

- 使用 Codex 辅助 STM32 裸机固件开发
- 使用 STM32CubeMX 生成初始化代码
- 使用 Keil MDK 管理和编译工程
- 使用 HAL 库进行外设开发
- 需要保护 `USER CODE BEGIN` / `USER CODE END` 区域
- 需要规范 AI 生成的驱动、算法和业务逻辑代码
- 需要统一 OLED、按键、串口、滤波、错误处理等常见模块设计

## 核心原则

- 固件工程由 STM32CubeMX 负责底层外设初始化
- Codex 只负责高层业务逻辑、驱动模块和算法实现
- 严禁破坏 CubeMX 自动生成结构
- `main.c` 保持干净，复杂逻辑放入独立模块
- 禁止使用 `HAL_Delay()` 实现业务等待
- 使用 `HAL_GetTick()` 编写非阻塞状态机
- 串口输入使用中断方式
- 传感器数据必须配套滤波算法
- OLED UI 独立成 `ui.c` / `ui.h` 模块
- 按键交互固定使用 PB12~PB15 四键布局
- STM32 与 ESP01S/Arduino 通信使用 JSON Lines 协议
- 中文说明与中文注释使用 UTF-8 编码

## 目录结构

```text
.
├── AGENTS.md
└── rules
    └── specs
        ├── coding_style_spec.md
        ├── comment_spec.md
        ├── comm_protocol_spec.md
        ├── error_handling_spec.md
        ├── filter_spec.md
        ├── key_spec.md
        └── oled_ui_spec.md
```

## 文件说明

| 文件 | 作用 |
| --- | --- |
| `AGENTS.md` | Codex 总规则入口，定义角色、协作边界、CubeMX 保护规则和编码约束 |
| `coding_style_spec.md` | C/HAL 编码风格、非阻塞写法、返回值检查等规范 |
| `comment_spec.md` | 中文注释、英文注释、Keil 兼容注释格式规范 |
| `comm_protocol_spec.md` | STM32 与 ESP01S/Arduino 的串口协议、JSON Lines、ACK/NACK 和重传规则 |
| `error_handling_spec.md` | 错误码、错误等级、错误上报、降级运行和看门狗边界 |
| `filter_spec.md` | 传感器数据滤波算法和参数建议 |
| `key_spec.md` | PB12~PB15 四键布局、中断消抖和按键事件规范 |
| `oled_ui_spec.md` | 0.96 寸 128x64 OLED UI 页面、显存刷新和交互规范 |

## 使用方式

将本规则包复制到 STM32 工程根目录，或将 `AGENTS.md` 与 `rules/specs` 放入你的 Codex 工作目录中：

```text
your-stm32-project
├── AGENTS.md
├── rules
│   └── specs
├── Core
├── Drivers
├── MDK-ARM
└── your_project.ioc
```

然后在 Codex 中打开该工程目录。Codex 在执行 STM32 相关开发任务时，会优先读取 `AGENTS.md`，并在涉及具体场景时按需读取 `rules/specs` 中的细分规范。

## 推荐工作流

1. 由开发者使用 STM32CubeMX 完成芯片、时钟、GPIO、UART、I2C、ADC、DMA 等底层配置。
2. 生成 Keil 工程后，将本规则包放入工程根目录。
3. 让 Codex 读取 `.ioc`、`main.c`、外设初始化文件和业务模块。
4. 需要新增传感器、OLED、按键、串口协议或控制算法时，让 Codex 按规则编写独立模块。
5. `main.c` 中只保留高层初始化和循环调度调用。
6. 修改后使用 Keil 编译，并根据编译错误继续修正。

## CubeMX 保护规则

本规则包默认固定面向：

- STM32CubeMX 6.3.0
- STM32CubeMX DB.6.0.30
- Keil MDK
- STM32 HAL 裸机工程

任何对 `main.c` 的修改都必须遵守：

- 只能修改 `/* USER CODE BEGIN */` 与 `/* USER CODE END */` 之间的内容
- `#include` 放在 `USER CODE BEGIN Includes`
- 宏定义放在 `USER CODE BEGIN PD`
- 私有变量放在 `USER CODE BEGIN PV`
- 函数声明放在 `USER CODE BEGIN PFP`
- 用户函数放在 `USER CODE BEGIN 4`
- 不手写 `SystemClock_Config`
- 不手写 `MX_GPIO_Init`
- 不擅自修改 CubeMX 自动生成的初始化代码

## 适合谁使用

- 使用 Codex 写 STM32 裸机代码的开发者
- 希望把 AI 生成代码限制在安全边界内的嵌入式开发者
- 使用 STM32CubeMX + Keil + HAL 的学生、毕业设计项目和小型硬件项目
- 需要统一 OLED、按键、串口、传感器滤波等基础模块规范的团队

## 不适用场景

- 完整 RTOS 架构规范
- Linux 驱动开发
- STM32CubeIDE 专用工程规范
- IAR 专用工程规范
- 已经深度封装自研 BSP/OSAL 的大型项目

这些场景可以基于本规则包改造，但不建议直接照搬。

## 开源建议

如果你基于本规则包创建自己的规则集，建议同步修改：

- CubeMX 版本
- 目标芯片系列
- 按键引脚定义
- OLED 驱动接口
- 串口协议字段
- 云平台字段
- 错误码表
- 滤波参数

## License

本项目采用 [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International](https://creativecommons.org/licenses/by-nc-sa/4.0/) 协议发布，即 `CC BY-NC-SA 4.0`。

你可以自由学习、复制、修改和分享本规则包，但必须保留署名，且不得用于商业用途。基于本项目修改后的版本也必须使用相同或兼容的非商业协议发布。
