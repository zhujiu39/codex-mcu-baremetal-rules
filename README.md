<div align="center">

<h1>MCU Bare-Metal Rules</h1>

<p><strong>面向通用 MCU 裸机与轻量固件项目的 AI Agent 开发规则包。</strong></p>

<p>适用于 STM32、GD32、Nuvoton 等不同芯片与厂商 SDK，重点约束工程事实确认、生成代码保护、非阻塞设计、外设边界、通信、错误处理、采样滤波和显示交互。</p>

<p>
  <a href="https://github.com/zhujiu39/codex-mcu-baremetal-rules/blob/main/LICENSE">
    <img alt="License" src="https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-blue.svg">
  </a>
  <a href="https://github.com/zhujiu39/codex-mcu-baremetal-rules">
    <img alt="GitHub Repo stars" src="https://img.shields.io/github/stars/zhujiu39/codex-mcu-baremetal-rules?style=social">
  </a>
  <a href="https://github.com/zhujiu39/codex-mcu-baremetal-rules">
    <img alt="GitHub forks" src="https://img.shields.io/github/forks/zhujiu39/codex-mcu-baremetal-rules?style=social">
  </a>
  <a href="https://github.com/zhujiu39/codex-mcu-baremetal-rules/commits/main">
    <img alt="Last commit" src="https://img.shields.io/github/last-commit/zhujiu39/codex-mcu-baremetal-rules">
  </a>
</p>

<p>
  <a href="#项目定位">项目定位</a> |
  <a href="#快速开始">快速开始</a> |
  <a href="#规则模块">规则模块</a> |
  <a href="#核心约束">核心约束</a> |
  <a href="#推荐工作流">推荐工作流</a> |
  <a href="#许可证">许可证</a>
</p>

</div>

---

## 项目定位

`MCU Bare-Metal Rules` 是一套可复制到 MCU 固件工程中的 AI Agent 规则包。它不提供固定代码实现，也不绑定某一家芯片、SDK 或 IDE。

规则的目标是让 Agent 在动手编码前先确认工程事实，再按当前项目选择实现方式：

- 当前使用什么芯片、SDK、BSP、IDE 和构建系统。
- 哪些文件或代码块由配置工具生成，哪些位置允许人工修改。
- 当前工程已经有哪些驱动封装、时间基准、错误处理、通信和 UI 模块。
- 引脚、通道、中断、DMA、时钟和外设资源来自哪里。
- 缺少硬件或协议信息时，应先询问用户，而不是套用示例。

## 适用场景

| 场景 | 说明 |
| --- | --- |
| MCU 裸机项目 | 主循环、状态机、轻量调度器或简单任务表 |
| 多厂商芯片项目 | STM32、GD32、Nuvoton 或其他常见 MCU |
| 厂商 SDK/BSP 工程 | 使用厂商库、寄存器封装、图形化配置工具或手写初始化 |
| 传感器与控制项目 | 采样、滤波、阈值、执行机构、显示和通信 |
| AI Agent 协作开发 | 给 Codex、Claude Code、OpenCode 等工具提供工程边界 |

## 快速开始

你可以用两种方式把本规则包接入 MCU 工程：

| 方式 | 适合场景 |
| --- | --- |
| Agent 自动安装 | 想让 AI 编程工具自动下载、复制、备份和验收规则文件 |
| 手动安装 | 想自己复制 `AGENTS.md` 和 `rules/specs`，或需要离线使用 |

接入完成后的工程结构通常类似：

```text
your-mcu-project
├── AGENTS.md
├── rules
│   └── specs
├── Core / App / User / Src
├── Drivers / BSP / Libraries
└── project files
```

目录名称不强制固定。Agent 应根据当前工程真实结构判断入口和可编辑区域。

## Agent 自动安装

本仓库提供了可直接复制给 AI 编程工具的自动安装 Prompt。用户只需要在目标 MCU 工程目录中打开对应工具，然后复制对应文件中的整段指令。

| 工具 | 使用文件 | 安装结果 |
| --- | --- | --- |
| Codex | [`how to use for codex.md`](how%20to%20use%20for%20codex.md) | 自动复制 `AGENTS.md` 和 `rules/specs` |
| Claude Code | [`how to use for claudecode.md`](how%20to%20use%20for%20claudecode.md) | 自动生成 `CLAUDE.md`，并复制 `rules/specs` |
| OpenCode | [`how to use for opencode.md`](how%20to%20use%20for%20opencode.md) | 自动复制 `AGENTS.md` 和 `rules/specs` |

自动安装 Prompt 会要求 Agent 执行以下动作：

- 判断当前目录是否像 MCU 固件工程根目录。
- 临时 clone 本规则仓库。
- 复制规则入口文件和 `rules/specs`。
- 遇到已有规则文件时先备份。
- 验收规则文件是否存在。
- 读取入口规则文件，并在后续开发中遵守规则。

## 手动安装

先 clone 本仓库：

```powershell
git clone https://github.com/zhujiu39/codex-mcu-baremetal-rules.git
```

进入仓库后，将 `AGENTS.md` 和 `rules` 复制到你的 MCU 工程根目录：

```powershell
Copy-Item .\AGENTS.md <your-mcu-project>\AGENTS.md
Copy-Item .\rules <your-mcu-project>\rules -Recurse
```

如果目标工程已经存在同名文件，建议先手动备份：

```powershell
Copy-Item <your-mcu-project>\AGENTS.md <your-mcu-project>\AGENTS.local-backup.md
Copy-Item <your-mcu-project>\rules <your-mcu-project>\rules.local-backup -Recurse
```

手动安装后至少确认以下文件存在：

```text
AGENTS.md
rules/specs/coding_style_spec.md
rules/specs/comment_spec.md
rules/specs/comm_protocol_spec.md
rules/specs/error_handling_spec.md
rules/specs/filter_spec.md
rules/specs/key_spec.md
rules/specs/oled_ui_spec.md
```

## 规则模块

| 文件 | 作用 |
| --- | --- |
| [`AGENTS.md`](AGENTS.md) | 总规则入口，定义角色、工程事实确认、生成代码保护和编码边界 |
| [`coding_style_spec.md`](rules/specs/coding_style_spec.md) | 通用 C 编码、非阻塞调度、外设调用、微秒时序和看门狗原则 |
| [`comment_spec.md`](rules/specs/comment_spec.md) | 注释与接口说明规范，避免重型模板化注释 |
| [`comm_protocol_spec.md`](rules/specs/comm_protocol_spec.md) | 外部通信、帧边界、缓冲区所有权、超时、重试和字段校验 |
| [`error_handling_spec.md`](rules/specs/error_handling_spec.md) | 错误分类、显式处理、降级运行、复位诊断和看门狗配合 |
| [`filter_spec.md`](rules/specs/filter_spec.md) | 采样链路、滤波算法选择、参数来源、阈值判断和 ADC 数据处理 |
| [`key_spec.md`](rules/specs/key_spec.md) | 按键、旋钮、触摸等人机输入的消抖、事件和交互边界 |
| [`oled_ui_spec.md`](rules/specs/oled_ui_spec.md) | OLED、LCD、数码管等显示界面的刷新、布局和模块边界 |

## 核心约束

| 约束 | 要求 |
| --- | --- |
| 先确认工程事实 | 编码前识别芯片、SDK、BSP、构建系统、生成代码边界和已有封装 |
| 不套厂商 API | 不假设某一家库函数、外设句柄、寄存器宏或工程结构存在 |
| 保护生成代码 | 工具生成文件只在允许区域或扩展文件中修改 |
| 主循环保持清晰 | 主循环只调度高层任务，复杂逻辑放入模块 |
| 非阻塞设计 | 业务等待、采样、刷新和通信重试都必须可退出、可超时 |
| 外设失败显式处理 | 外设调用、协议解析、参数入口和缓冲区操作必须检查失败路径 |
| 硬件信息有来源 | 引脚、通道、中断、DMA、时钟和外设资源必须来自工程或用户确认 |
| 跨芯片可移植 | 切换芯片、SDK、主频、外设映射或显示硬件后必须重新确认相关假设 |

## 推荐工作流

1. 打开目标 MCU 工程，先让 Agent 读取入口规则。
2. Agent 检查工程文件、构建配置、SDK/BSP、生成代码边界和已有模块。
3. 涉及硬件资源时，Agent 从工程配置、原理图说明、头文件或用户输入确认来源。
4. 新增功能时，先选择与当前工程一致的驱动层和模块边界。
5. 编写最小必要改动，并保持生成代码和已有风格不被破坏。
6. 运行当前工程已有的编译、lint、静态检查或 smoke test。
7. 用户审查 diff 后再决定是否提交。

## 不适用场景

| 场景 | 原因 |
| --- | --- |
| Linux 驱动开发 | 本规则面向 MCU 固件，不覆盖 Linux 内核模型 |
| 完整 RTOS 架构设计 | 本规则只覆盖裸机或轻量调度边界 |
| 高安全认证流程 | 功能安全、医疗、车规等认证需要额外流程和证据链 |
| 通用应用层软件 | 规则强关注 MCU 外设、硬件资源和固件约束 |

## 自定义建议

如果你基于本规则包创建自己的项目规则，可以按项目补充：

- 芯片系列、SDK 和 IDE。
- 生成代码边界和允许修改区域。
- 板级资源表、引脚表和原理图约束。
- 通信协议、错误码和日志接口。
- 采样周期、滤波参数和标定流程。
- 显示与输入交互设计。

这些内容建议写在具体工程自己的规则文件中，不放回通用规则包。

## Star History

<a href="https://www.star-history.com/#zhujiu39/codex-mcu-baremetal-rules&type=timeline&logscale&legend=top-left">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/chart?repos=zhujiu39/codex-mcu-baremetal-rules&type=timeline&logscale&theme=dark&legend=top-left" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/chart?repos=zhujiu39/codex-mcu-baremetal-rules&type=timeline&logscale&legend=top-left" />
   <img alt="Star History Chart" src="https://api.star-history.com/chart?repos=zhujiu39/codex-mcu-baremetal-rules&type=timeline&logscale&legend=top-left" />
 </picture>
</a>

## 许可证

本项目采用 [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International](https://creativecommons.org/licenses/by-nc-sa/4.0/) 协议发布，即 `CC BY-NC-SA 4.0`。

你可以自由学习、复制、修改和分享本规则包，但必须保留署名，且不得用于商业用途。基于本项目修改后的版本也必须使用相同或兼容的非商业协议发布。

商业使用需要获得作者额外授权。
