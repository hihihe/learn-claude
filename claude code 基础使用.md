## 一、 项目启动

**核心要点：Claude Code 是基于当前目录**上下文运行的，它在生成 `CLAUDE.md` 时会参考整个代码库。**切勿在混杂的根目录下直接运行**，否则会导致上下文混乱和逻辑冲突。

**正确操作流程：**

1. **独立文件夹**：为每一个项目创建一个独立的文件夹。
2. **启动**：**进入**项目**目录**后，输入 `claude` 启动。
3. **初始化**：首次运行输入 `/init`，系统会扫描代码库并在项目根目录生成 `CLAUDE.md` 配置文件。

### 无中断模式启动（YOLO Mode）

- **启动命令**：`claude --dangerously-skip-permissions`
- **用途**：让 Claude 一口气干完活，跳过所有权限确认。适合批量修复或生成样板代码。
- **⚠️ 安全警告**：此模式有风险！可能导致数据丢失、系统损坏或数据泄露。**强烈建议在 Docker 等隔离容器里使用以降低风险。**

## 二、 核心交互机制：模式与记忆

1. ### 交互模式切换

使用快捷键 `Alt+M` 或 `Shift+Tab` 在三种模式间循环切换：

- **普通模式（Normal Mode）**：默认模式，适合常规交互。AI 提议操作，用户确认后执行。
- **计划模式（Plan Mode）**：**推荐初期使用**。此模式下，AI 只会与您讨论和规划，**不会执行代码**。非常适合在项目初期进行需求沟通和方案设计。
- **接受模式（Accept Edit Mode）**：AI 自动接受会话的文件编辑权限。

> **最佳实践**：初期建议使用**计划模式**，与 AI 充分讨论需求、调整方案。确认无误后，再切换至普通或自动接受模式执行任务。

1. ### 管理项目记忆（`CLAUDE.md`）

`CLAUDE.md` 是 Claude 的"长期记忆"向导，记录了重要命令、项目架构和编码风格。每次启动 Claude Code 时，该文件内容会自动加载到上下文中。

- **创建方式**：
  - **自动生成**：运行 `/init`。
  - **手动编辑**：直接编辑项目根目录的 `CLAUDE.md` 文件。
- **查看**：运行 `/memory` 命令查看。

1. ### 上下文管理

上下文是 AI 的核心资源。Claude Code 提供多种命令来有效管理。

- `/context`：**查看上下文使用情况**，了解当前上下文占用和剩余空间。
- `/compact`：**压缩上下文**，去除无关信息，保留核心摘要。可在上下文即将用尽时（剩余 20%）主动使用。
- `/clear`：**清空上下文**，清除所有对话历史，开始全新会话（不会删除 `CLAUDE.md`）。
- `/export`：导出当前会话为 Markdown 文件，方便后续参考。
- `/resume`：列出所有历史聊天记录，选择后继续对话。

## 三、 高阶工作流：Spec 文档驱动开发

**核心理念**：文档，文档，还是文档！清晰的文档是确保 AI 开发结果符合预期的关键。不要上来就写代码。

Spec (Specification, 规格) 工作流通过一份规格文档来驱动 AI，这份文档是 AI 必须遵守的唯一契约。

### Spec 四部曲流程

1. **规范（Specification）**
   1. **动作**：描述项目功能、目标用户、核心逻辑。
   2. **目标**：将模糊想法转化为结构化需求。您只需提供想法，让 AI 生成规范文档。
2. **计划（Plan）**
   1. **动作**：切换到**计划模式（Plan Mode）**。
   2. **目标**：将生成的规范文档，结合您的技术偏好（框架、语言）和约束，让 AI 生成完整的技术方案。
3. **任务（Task）**
   1. **动作**：让 AI 参考前两份文档。
   2. **目标**：**将功能需求拆解为可独立实现和测试的具体子任务列表**。
4. **实现（Implementation）**
   1. **动作**：AI 会将以上所有内容汇总成一个 `spec.md` 文件。
   2. **执行**：您确认无误后，切换回**自动接受模式（Auto-Accept Mode）**，指令"按照 `spec.md` 开始执行任务"，它便会开始编写代码。

> **Spec vs.** `CLAUDE.md` **区别**：
>
> - `spec.md` 侧重于当前具体功能的详细实现计划。
> - `CLAUDE.md` 则更宏观，关注整体偏好、项目规则和长期记忆。

### 个人工作流习惯参考

1. **撰写初稿**：自己先写一份大致的需求文档。
2. **讨论优化**：发给 AI，基于文档进行讨论，寻求建议。
3. **确认理解**：让 AI 复述确认后的信息，确保双方理解一致。
4. **生成终稿**：让 AI 将确认后的所有信息（包含技术方案、任务拆分等）更新到文档中，生成最终 Spec。
5. **开始执行**：将最终文档发给 AI，让它开始开发，您只需等待测试即可。

## 四、 扩展能力：思考、代理与插件

1. ### 深度思考模式（Thinking）

在处理复杂任务（如架构重构、调试疑难问题）时激活。

- **触发方式**：
  - 提示词关键词：`ultrathink`（设置该轮思考努力程度为 `high`，消耗更多 Token）。
  - 快捷键：`Alt+T`（Win/Linux）或 `Option+T`（Mac），切换 Thinking on/off。
  - 在提示词中使用 `effort: max` 设置最高思考级别。

> **注意**：`think`、`think hard`、`think more` 等普通提示词**不会**分配思考 token，仅作为常规指令被理解。只有 `ultrathink` 和 effort 级别设置才能真正激活深度思考。

1. ### MCP（模型上下文协议）

[Claude Code MCP安装](https://ai.feishu.cn/docx/YxhFdJfbToSWqoxiPYZc9Zbrnaf)

通过 `/mcp` 管理扩展工具，赋予 Claude 联网和操作能力。

- **[Context7（必装）](https://github.com/upstash/context7)**：
  - **功能**：让 Claude 能获取最新的代码知识，解决大模型知识库过时的问题。它直接从源头提取最新的、特定版本的文档和代码示例。
  - **资源**：[安装指南](https://context7.com/docs/installation) | [仪表板](https://context7.com/dashboard)
- **[Playwright](https://github.com/microsoft/playwright)**：
  - **功能**：浏览器自动化，主要用于进行**端到端测试**和网页抓取。
- **[Firecrawl](https://www.firecrawl.dev/)**：
  - **功能**：抓取网页内容并转换为 Markdown。
- **[Browser Use](https://github.com/browser-use/browser-use)**：
  - **功能**：让 AI Agent 自主控制浏览器，支持点击、输入、滚动等操作，适合网页交互和数据采集场景。

> **注意：购买国内模型厂商提供的编程会员，也需要安装指定的 MCP 服务，才能实现图像理解、搜索等功能。**

## 五、 指令与快捷键速查手册

### 核心命令 (Slash Commands)

| 命令        | 描述                                           |
| ----------- | ---------------------------------------------- |
| /init       | 初始化项目，扫描代码并生成 CLAUDE.md。         |
| /clear      | 清除当前会话的对话历史（重置上下文）。         |
| /compact    | 压缩上下文，总结对话历史，释放 Token 空间。    |
| /resume     | 列出所有历史聊天记录，选择后继续对话。         |
| /rewind     | 回退对话/代码，弹出菜单选择：回退代码+对话 / 仅对话 / 仅代码 / 从此压缩 / 取消。 |
| /memory     | 查看和管理 CLAUDE.md 记忆内容。                |
| /agents     | 管理子代理 (Subagents)。                       |
| /mcp        | 管理 MCP 服务器连接，检查可用工具。            |
| /permissions| 确认功能及工具权限设置。                       |
| /config     | 进入配置栏（主题、权限等）。                   |
| /model      | 切换语言模型。                                 |
| /cost       | 查看当前会话费用情况。                         |
| /context    | 查看上下文使用情况（彩色网格可视化）。         |
| /export     | 导出完整聊天记录为 Markdown 文件。             |
| /btw        | 不中断当前任务的前提下提问，不污染上下文。     |
| /branch     | 分叉当前对话（`/fork` 为别名）。               |
| /insights   | 生成月度使用分析报告。                         |
| /rc         | 远程控制，手机操作终端会话（`/remote-control`）。 |
| /help       | 查看所有可用命令。                             |
| exit        | 退出 Claude Code。                             |

### 内置 Skills

| 命令       | 描述                                           |
| ---------- | ---------------------------------------------- |
| /simplify  | 三角度代码审查（复用、质量、效率），并行 Agent。 |
| /loop      | 定时重复执行任务，如 `/loop 5m 检查部署状态`。  |

### 实用命令详解

#### `/btw` — 不中断提问

Claude 正在干活时，突然想到一个问题？直接输入 `/btw`，它会在**独立进程**中回答，不会中断当前任务，也不会污染对话历史。回答完按空格或回车即可消除。几乎不消耗额外 Token（复用当前提示缓存）。

#### `/branch` — 对话分叉

与 `/rewind`（回退）不同，`/branch` 是**分叉**——从当前节点复制出一条新的对话分支，原会话不受影响。适合想同时尝试两种方案时使用：一个会话走方案 A，另一个走方案 B，最后挑效果好的。`/fork` 是其别名。

#### `/rc` — 手机远程控制

输入 `/rc` 或 `/remote-control`，生成一个 URL，手机打开即可同步操作终端会话。两端完全同步，可交替使用。代码始终在电脑上运行，手机只是遥控器，文件系统、MCP 服务器、项目配置全部在本地。

#### `/insights` — 使用分析报告

生成一份本地 HTML 报告，分析过去一个月的使用习惯：常用命令、重复操作模式、推荐自定义命令和 Skills。建议每月运行一次。

#### `/loop` — 定时循环任务

让 Claude 定时重复执行某个任务。用法：`/loop <时间间隔> <任务>`，默认间隔 10 分钟。例如：`/loop 5m 检查部署状态`。注意：定期任务创建 3 天后自动过期并删除。

#### `/model opusplan` — Pro 用户省钱模式

隐藏模式：需要深度推理时自动以 plan 模式使用 Claude Opus 4.6，切换到 Claude Sonnet 4.6 进行实际执行。规划用 Opus（理解架构和依赖），写代码用 Sonnet（快且够用），两全其美。适合 Pro 订阅用户节省 Opus 额度。

#### `/export` — 导出对话

将当前整段对话导出为 Markdown 文件。适合保存架构讨论、方案推敲等重要内容，也可导出后交给其他工具（如 Codex）协同使用。

### 快捷键与特殊符号

| 符号/键           | 描述                                                         |
| ----------------- | ------------------------------------------------------------ |
| Alt+M / Shift+Tab | 切换工作模式 (Normal / Plan / Auto-Accept)。                 |
| Alt+T / Option+T  | 切换思考模式开/关（Thinking on/off）。                       |
| ESC               | 单按：打断 Claude 当前操作。                                 |
| ESC ESC           | 连按两次：回退对话（等同于 /rewind）。                       |
| @                 | 提及文件。例如 @main.py，将文件内容包含在上下文中。          |
| !                 | 执行 Bash 命令。例如 !ls -la，在命令前添加 ! 可执行常规终端命令。 |
| Ctrl+V            | 粘贴图片，让 Claude 理解图像内容。（Mac iTerm2 下 Cmd+V 也可用。） |
| Ctrl+J / Option+Enter | 输入框换行。                                  |
| Ctrl+R            | 搜索历史 Prompt 记录。                         |
| Ctrl+U            | 删除输入框整行内容。                           |

### 换行输入方法

在 Claude Code 中输入多行内容时，有以下几种方式：

**方法一：快捷方式（所有终端通用）**

输入 `\` 然后按 Enter。

**方法二：自动配置**

在 Claude Code 会话中运行 `/terminal-setup`，会自动为你配置 `Shift+Enter` 或 `Shift+Ctrl` 作为换行快捷键。

**方法三：键盘快捷键**

| 快捷键       | 平台/终端     |
| ------------ | ------------- |
| Option+Enter | macOS（默认） |
| Ctrl+J       | 所有平台       |

## 附：参考链接

[快速开始](https://docs.anthropic.com/zh-CN/docs/claude-code/quickstart)

https://www.deeplearning.ai/short-courses/claude-code-a-highly-agentic-coding-assistant/

[Prompts & Summaries of Lessons](https://github.com/https-deeplearning-ai/sc-claude-code-files/tree/main/reading_notes)

[Claude Code 常见工作流程](https://docs.anthropic.com/zh-CN/docs/claude-code/common-workflows)

[Claude Code 最佳实践](https://www.anthropic.com/engineering/claude-code-best-practices)

[Claude Code 用例](https://www.anthropic.com/news/how-anthropic-teams-use-claude-code)

[Claude Code 实际应用](https://anthropic.skilljar.com/claude-code-in-action)

[How I Use Every Claude Code Feature](https://blog.sshh.io/p/how-i-use-every-claude-code-feature)

[How I Use Every Claude Code Feature 翻译版](https://mp.weixin.qq.com/s/bSGFZCUWgi_fv_T4lHOsnQ)