<div align="center">

<h1>Codex STM32 Bare-Metal Rules</h1>

<p><strong>Codex rules for reliable STM32 bare-metal firmware development.</strong></p>

<p>为 <code>STM32CubeMX 6.3.0</code> + <code>Keil MDK</code> + <code>STM32 HAL</code> 工程提供清晰、可执行、可复用的 AI Agent 开发边界。</p>

<p>
  <a href="https://github.com/zhujiu39/codex-stm32-baremetal-rules/blob/main/LICENSE">
    <img alt="License" src="https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-blue.svg">
  </a>
  <a href="https://github.com/zhujiu39/codex-stm32-baremetal-rules">
    <img alt="GitHub Repo stars" src="https://img.shields.io/github/stars/zhujiu39/codex-stm32-baremetal-rules?style=social">
  </a>
  <a href="https://github.com/zhujiu39/codex-stm32-baremetal-rules">
    <img alt="GitHub forks" src="https://img.shields.io/github/forks/zhujiu39/codex-stm32-baremetal-rules?style=social">
  </a>
  <a href="https://github.com/zhujiu39/codex-stm32-baremetal-rules/commits/main">
    <img alt="Last commit" src="https://img.shields.io/github/last-commit/zhujiu39/codex-stm32-baremetal-rules">
  </a>
</p>

<p>
  <a href="#快速开始">快速开始</a> |
  <a href="#规则模块">规则模块</a> |
  <a href="#核心约束">核心约束</a> |
  <a href="#推荐工作流">推荐工作流</a> |
  <a href="#许可证">许可证</a>
</p>

</div>

---

## 项目定位

`Codex STM32 Bare-Metal Rules` 提供了一套面向 STM32 裸机固件开发的 Codex 规则体系，用于提升 AI Agent 参与嵌入式项目时的代码可靠性、工程一致性和 CubeMX 兼容性。

这些规则围绕真实 STM32 开发痛点设计：保护 `USER CODE` 区域、避免阻塞式业务等待、规范 HAL 返回值检查、拆分高层模块、统一传感器滤波、OLED UI、按键消抖和 ESP01S 串口通信协议。

本仓库不是一个完整固件工程，而是一个可复制到 STM32 项目根目录的 Agent Rules 模板。

## 适用场景

| 场景 | 说明 |
| --- | --- |
| Codex 辅助开发 | 让 Codex 在明确边界内编写 STM32 裸机业务代码 |
| CubeMX + Keil 工程 | 保护 CubeMX 自动生成结构，避免破坏初始化代码 |
| HAL 裸机项目 | 约束 HAL 调用、非阻塞调度、返回值检查和错误处理 |
| 传感器项目 | 统一滤波、串口上报、OLED 显示和按键交互规范 |
| 毕设/课程设计 | 给常见 STM32 小型项目提供可复用的 AI 开发规则 |

## 快速开始

将本仓库中的 `AGENTS.md` 和 `rules/specs` 复制到你的 STM32 工程根目录：

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

然后在 Codex 中打开该工程目录。Codex 会优先读取 `AGENTS.md`，并在涉及具体开发场景时按需读取 `rules/specs` 下的细分规范。

```powershell
git clone https://github.com/zhujiu39/codex-stm32-baremetal-rules.git
```

如果你只想在现有工程中使用规则，可以复制：

```powershell
Copy-Item .\AGENTS.md <your-stm32-project>\AGENTS.md
Copy-Item .\rules <your-stm32-project>\rules -Recurse
```

## 规则模块

| 文件 | 作用 |
| --- | --- |
| [`AGENTS.md`](AGENTS.md) | Codex 总规则入口，定义角色、协作边界、CubeMX 保护规则和编码约束 |
| [`coding_style_spec.md`](rules/specs/coding_style_spec.md) | C/HAL 编码风格、非阻塞写法、返回值检查等规范 |
| [`comment_spec.md`](rules/specs/comment_spec.md) | 中文注释、英文注释、Keil 兼容注释格式规范 |
| [`comm_protocol_spec.md`](rules/specs/comm_protocol_spec.md) | STM32 与 ESP01S/Arduino 的串口协议、JSON Lines、ACK/NACK 和重传规则 |
| [`error_handling_spec.md`](rules/specs/error_handling_spec.md) | 错误码、错误等级、错误上报、降级运行和看门狗边界 |
| [`filter_spec.md`](rules/specs/filter_spec.md) | 传感器数据滤波算法和参数建议 |
| [`key_spec.md`](rules/specs/key_spec.md) | PB12~PB15 四键布局、中断消抖和按键事件规范 |
| [`oled_ui_spec.md`](rules/specs/oled_ui_spec.md) | 0.96 寸 128x64 OLED UI 页面、显存刷新和交互规范 |

## 核心约束

| 约束 | 要求 |
| --- | --- |
| CubeMX 保护 | 严禁修改 `USER CODE BEGIN` / `USER CODE END` 之外的自动生成代码 |
| `main.c` 简洁 | `while(1)` 中只调用高层接口，复杂逻辑放独立模块 |
| 非阻塞调度 | 禁止用 `HAL_Delay()` 实现业务等待，统一使用 `HAL_GetTick()` |
| HAL 返回值 | 阻塞 HAL 调用必须检查返回值 |
| 静态内存 | 禁止 `malloc`，优先使用静态缓冲区 |
| 串口输入 | 必须使用中断方式接收 |
| 串口上报 | 默认 3 秒 1 次，使用 `ESP_REPORT_INTERVAL_MS` 限速 |
| 传感器数据 | 必须配套滤波算法 |
| OLED UI | 独立 `ui.c` / `ui.h`，使用显存缓冲刷新 |
| 按键交互 | 固定 PB12~PB15 四键布局，使用中断消抖 |

## CubeMX 兼容性

本规则包默认面向以下工程组合：

| 项目 | 默认版本/工具 |
| --- | --- |
| STM32CubeMX | `6.3.0` |
| CubeMX DB | `DB.6.0.30` |
| IDE | Keil MDK |
| 驱动库 | STM32 HAL |
| 系统模型 | 裸机循环调度 |

如需适配 STM32CubeIDE、IAR、FreeRTOS 或自研 BSP/OSAL，建议基于本规则包单独派生一份项目规则。

## 推荐工作流

1. 张三使用 STM32CubeMX 完成芯片、时钟、GPIO、UART、I2C、ADC、DMA 等底层配置。
2. CubeMX 生成 Keil 工程。
3. 将本规则包复制到工程根目录。
4. Codex 读取 `.ioc`、`main.c`、外设初始化文件和业务模块。
5. 新增传感器、OLED、按键、串口协议或控制算法时，由 Codex 按规则编写独立模块。
6. `main.c` 中只保留高层初始化和循环调度调用。
7. 使用 Keil 编译，并根据编译结果继续修正。

## 不适用场景

| 场景 | 原因 |
| --- | --- |
| Linux 驱动开发 | 本规则只覆盖 STM32 裸机工程 |
| 完整 RTOS 架构 | 当前规则以裸机循环调度为主 |
| 大型自研 BSP | 需要结合项目自身架构重写规则 |
| 通用 C 项目 | 规则强绑定 CubeMX、Keil、HAL 和 STM32 外设模型 |

## 开源前自定义建议

如果你基于本规则包创建自己的规则集，建议同步修改：

- CubeMX 版本
- 目标芯片系列
- 按键引脚定义
- OLED 驱动接口
- 串口协议字段
- 云平台字段
- 错误码表
- 滤波参数
- 协作角色名称

## 许可证

本项目采用 [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International](https://creativecommons.org/licenses/by-nc-sa/4.0/) 协议发布，即 `CC BY-NC-SA 4.0`。

你可以自由学习、复制、修改和分享本规则包，但必须保留署名，且不得用于商业用途。基于本项目修改后的版本也必须使用相同或兼容的非商业协议发布。

商业使用需要获得作者额外授权。
