# How to Use for Claude Code

把下面整段内容复制给 Claude Code。Claude Code 会在当前 STM32 工程目录中自动接入本规则包。

```text
你现在要为当前 STM32 工程自动接入 `Codex STM32 Bare-Metal Rules`。

规则仓库：
https://github.com/zhujiu39/codex-stm32-baremetal-rules

目标：
1. 将规则仓库中的 `AGENTS.md` 转换并写入当前工程根目录的 `CLAUDE.md`。
2. 将规则仓库中的 `rules/specs` 复制到当前工程根目录的 `rules/specs`。
3. 不修改任何 STM32 源码、`.ioc`、Keil 工程文件或业务文件。
4. 若当前工程已有 `CLAUDE.md`、`AGENTS.md` 或 `rules`，先备份，不要直接覆盖。
5. 完成后读取并理解 `CLAUDE.md`，后续所有 STM32 开发都必须遵守其中规则。

请按下面流程执行：

1. 先判断当前目录是否像 STM32 工程根目录。
   - 优先查找 `.ioc`
   - 查找 `Core`
   - 查找 `Drivers`
   - 查找 `MDK-ARM`
   - 查找 `.uvprojx`
   如果当前目录不像 STM32 工程根目录，先停止，并告诉我应该切换到哪个目录。

2. 在临时目录下载规则仓库。
   - 优先使用：
     `git clone https://github.com/zhujiu39/codex-stm32-baremetal-rules.git`
   - 临时目录可放在系统 temp 目录或当前工程外部。
   - 不要把整个规则仓库嵌套到我的 STM32 工程里。

3. 复制并转换规则文件。
   - 读取规则仓库中的 `AGENTS.md`。
   - 将其内容写入当前工程根目录的 `CLAUDE.md`。
   - 写入时只需要把文件名从 `AGENTS.md` 改为 `CLAUDE.md`，规则正文不要改写、不要摘要、不要删减。
   - 复制 `rules/specs` 到当前工程根目录的 `rules/specs`。
   - 如果目标已存在 `CLAUDE.md`，先备份为：
     `CLAUDE.local-backup.md`
   - 如果目标已存在 `AGENTS.md`，不要删除，必要时备份为：
     `AGENTS.local-backup.md`
   - 如果目标已存在 `rules`，先备份为：
     `rules.local-backup`
   - 备份时如果同名备份已经存在，使用带时间戳的备份名。

4. 验收安装结果。
   必须确认这些文件存在：
   - `CLAUDE.md`
   - `rules/specs/coding_style_spec.md`
   - `rules/specs/comment_spec.md`
   - `rules/specs/comm_protocol_spec.md`
   - `rules/specs/error_handling_spec.md`
   - `rules/specs/filter_spec.md`
   - `rules/specs/key_spec.md`
   - `rules/specs/oled_ui_spec.md`

5. 安装完成后，读取 `CLAUDE.md`。
   后续你必须按 `CLAUDE.md` 的规则工作，尤其是：
   - 使用简体中文回复
   - PowerShell 读取中文文件使用 `Get-Content -Encoding utf8`
   - 不破坏 STM32CubeMX 自动生成结构
   - 只在 `USER CODE BEGIN` / `USER CODE END` 区域内修改 `main.c`
   - 不写 `SystemClock_Config` / `MX_GPIO_Init`
   - 不使用 `HAL_Delay()` 做业务等待
   - 串口输入使用中断
   - OLED、按键、滤波、通信协议等场景先读对应 `rules/specs` 文件

6. 最后给我一个简短结果：
   - 当前识别到的 STM32 工程路径
   - 已复制或生成的文件列表
   - 是否创建了备份
   - 是否成功读取 `CLAUDE.md`
   - 后续开发时需要注意的 3 条最高优先级规则

现在开始执行。不要只解释方案，直接操作。
```
