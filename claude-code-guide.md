# Claude Code 工作原理与扩展系统完整指南

> 基于 Claude Code 官方文档整理，最后更新：2026-03-25

---

## 全局概览

### 扩展基本功能

内置工具是基础。在此基础上，可通过 **Skills** 扩展 Claude 的知识库、**MCP** 连接外部服务、**Hooks** 自动化工作流、**Subagents** 委派独立任务。这些扩展构成核心代理循环之上的能力层。

### Claude 可以访问什么

本指南重点关注终端使用。Claude Code 也可在 VS Code、JetBrains IDE 等环境中运行。在目录中运行 `claude` 时，Claude Code 可以访问：

- **项目文件** — 工作目录及子目录下的所有文件，以及获得许可的其他位置文件
- **终端能力** — 可执行的任何命令：构建工具、git、包管理器、系统工具、脚本等
- **Git 状态** — 当前分支、未提交更改、最近提交历史
- **CLAUDE.md** — Markdown 文件，存储项目约定、编码规范等每个会话都应了解的上下文
- **自动记忆** — Claude 工作时自动保存的项目模式和偏好，MEMORY.md 前 200 行在每次会话开始时加载
- **已配置的扩展** — MCP Servers（外部服务）、Skills（工作流指令）、Subagents（任务委派）、Claude in Chrome（浏览器交互）

### 与内联代码助手的区别

Claude Code 看到**整个项目**，而非单个文件。当要求"修复身份验证错误"时，它会搜索相关文件、跨文件理解上下文、协调多处编辑、运行测试验证修复，并在需要时提交更改——这与只能看到当前文件的内联代码助手有本质区别。

---

## 一、整体架构

### 1.1 三阶段 Agentic Loop

Claude Code 的核心是一个**自主循环**，由模型自己驱动：

![Agentic Loop 三阶段自主循环](claude%20code%20images/Agentic%20Loop%20三阶段自主循环.jpg)

关键点：**模型自己决定何时收集上下文、何时行动、何时验证**，而不是按固定流程执行。

### 1.2 启动流程

![Claude Code 启动流程](claude%20code%20images/Claude%20Code%20启动流程.jpg)

> **注意**：S1-S5 的加载顺序按优先级从高到低排列，仅为了直观。实际流程是所有设置加载完毕后，在 S6 步骤**按优先级规则合并**（标量覆盖、数组拼接去重、deny 不可覆盖）。详见 1.6 设置优先级。

### 1.3 配置优先级 vs 上下文加载

**配置**和**记忆**是两套不同的系统，优先级机制完全不同：

| 类型 | 机制 | 冲突处理 | 适用范围 |
|------|------|---------|---------|
| **配置优先级** | 硬覆盖 | 高优先级**完全覆盖**低优先级 | settings.json、权限、hooks 等 |
| **上下文加载** | 软合并 | 全部加载拼接，冲突时 Claude **任意选择** | CLAUDE.md、MEMORY.md、rules 等 |

#### 配置优先级（硬性）

```
托管策略 > CLI 参数 > 本地设置 > 项目设置 > 用户设置
```

高优先级的配置**直接覆盖**低优先级。例如项目设置 `model: "sonnet"` 会覆盖用户设置的 `model: "haiku"`。deny 规则在任何层级都是绝对的。

#### 上下文加载（软性）

CLAUDE.md、MEMORY.md、rules 文件**全部加载**到上下文中，按顺序拼接：

```
托管 CLAUDE.md → 用户 CLAUDE.md → 项目 CLAUDE.md → CLAUDE.local.md → rules → MEMORY.md
```

如果两个文件规则冲突，Claude 可能任意选择其一。官方文档原文：
> "If two rules contradict each other, Claude may pick one arbitrarily."

**结论**：需要确定性执行的规则放**配置**（settings.json），需要 Claude 理解和遵循的指令放**上下文**（CLAUDE.md）。

### 1.4 上下文窗口管理

上下文窗口是有限资源，Claude Code 精细管理其分配：

| 组件 | 预算 |
|------|------|
| 对话历史 | 可变（达到 ~95% 时自动压缩） |
| CLAUDE.md 内容 (建议不超过200行) | 始终加载（压缩后重新注入） |
| 自动记忆 (MEMORY.md) | 前 200 行（超出部分保留在磁盘，会话中可主动读取，详见 3.5 节） |
| Skill 描述 | 动态缩放至 2% 上下文（fallback 16,000 字符） |
| Agent 描述 | 会话启动时加载（无官方公开的固定比例） |
| MCP 工具描述 | 最多 10%（超出则启用懒加载） |
| 工具搜索结果 | 按需加载 |

**自动压缩 (Auto-compaction)**：当上下文达到 ~95% 容量时，自动压缩保留关键信息，丢弃次要内容。CLAUDE.md 内容在压缩后始终重新注入。

### 1.5 权限模式

| 模式 | 行为 |
|------|------|
| `default` | 读取文件；编辑和命令需确认 |
| `acceptEdits` | 读取和编辑无需确认；命令仍需确认 |
| `plan` | 只读探索；不允许编辑 |
| `auto` | 所有操作经过后台安全分类器评估 |
| `dontAsk` | 仅执行预批准的工具；完全非交互 |
| `bypassPermissions` | 所有操作，无任何检查 |

### 1.6 设置优先级

设置从多个范围解析，**高优先级覆盖低优先级**：

| 优先级 | 来源 | 配置文件/位置 | 是否共享 |
|--------|------|-------------|---------|
| 1（最高） | 托管策略 - 服务器下发 | Anthropic 服务器推送 | 组织级 |
| 1（最高） | 托管策略 - MDM/OS 级 | macOS plist / Windows 注册表 | 组织级 |
| 1（最高） | 托管策略 - 本地文件 | `C:\Program Files\ClaudeCode\managed-settings.json`（Windows）<br/>`/Library/Application Support/ClaudeCode/managed-settings.json`（macOS）<br/>`/etc/claude-code/managed-settings.json`（Linux） | 组织级 |
| 2 | 命令行参数 | CLI flags，如 `--model`、`--allowedTools` | 仅当前会话 |
| 3 | 本地项目设置 | `.claude/settings.local.json`（项目根目录） | 否（gitignored） |
| 4 | 共享项目设置 | `.claude/settings.json`（项目根目录） | 是（提交到 Git） |
| 5（最低） | 用户设置 | `~/.claude/settings.json` | 否 |

> **关键规则**：托管策略**不可被任何级别覆盖**，包括命令行参数。同一时间只使用一个托管来源（服务器 > MDM > 本地文件 > 注册表），不会合并。

#### 合并行为

| 设置类型 | 合并方式 | 示例 |
|---------|---------|------|
| 标量（字符串/布尔/数字） | **高优先级覆盖低优先级** | 项目设置 `model: "sonnet"` 覆盖用户设置的 `model: "haiku"` |
| 数组（allow/deny 列表） | **拼接去重，不是替换** | 托管 allow `/opt/tools` + 用户 allow `~/.local` = 两个路径都生效 |

适用于数组合并的设置：`permissions.allow`、`permissions.deny`、`permissions.ask`、`sandbox.filesystem.allowWrite`、`sandbox.filesystem.denyWrite`、`sandbox.filesystem.denyRead`、`sandbox.filesystem.allowRead`、`allowedHttpHookUrls`、`httpHookAllowedEnvVars`。

#### Deny 规则不可覆盖

> "If a tool is denied at any level, no other level can allow it."

高层设置的 deny 是绝对的。比如托管策略 deny 了 `Bash(rm *)`，即使命令行参数 allow 了，也会被阻止。

#### 查看当前生效的设置

在 Claude Code 中运行 `/status` 可查看当前生效的所有设置及其来源。

---

## 二、六大扩展机制总览

| 机制 | 回答的问题 | AI 参与？ | 定义方式 | 类比 |
|------|-----------|----------|---------|------|
| **CLAUDE.md** | 始终遵守什么规则？ | 是 | Markdown | 团队手册 |
| **Skills** | Claude 该怎么做？ | 是 | SKILL.md | 操作手册 |
| **Subagents** | 谁来独立完成任务？ | 是（独立实例） | .md 文件 | 专项员工 |
| **MCP Servers** | Claude 能访问什么？ | 否（协议桥接） | .mcp.json | 外部工具箱 |
| **Hooks** | 什么时候自动执行？ | 部分 | settings.json | 自动触发器 |
| **Plugins** | 如何打包和分享以上所有？ | 否（纯打包） | plugin.json | 运输集装箱 |

![六大扩展机制全景图](claude%20code%20images/六大扩展机制全景图.jpg)

### 推荐采用路径

![推荐采用路径](claude%20code%20images/推荐采用路径.jpg)

---

## 三、CLAUDE.md — 项目指令

### 3.1 是什么

CLAUDE.md 是 Claude Code 在每次会话开始时加载的 Markdown 文件，包含项目指令、约定和上下文。它是**跨会话持久化知识**的主要机制。

### 3.2 文件位置与优先级

| 位置 | 范围 | 提交到 Git？ |
|------|------|-------------|
| 托管策略路径 | 组织级别 | 否 |
| `.claude/CLAUDE.md` 或 `CLAUDE.md` | 项目根目录 | 是 |
| `CLAUDE.local.md` | 项目根目录（本地） | 否 |
| `~/.claude/CLAUDE.md` | 用户主目录 | 否 |

> **注意**：用户级路径是 `~/.claude/CLAUDE.md`，不是 `~/CLAUDE.md`。

#### CLAUDE.md 与 CLAUDE.local.md 的关系

两者是**合并（拼接）关系**，不是覆盖。`CLAUDE.local.md` 在 `CLAUDE.md` 之后加载，内容全部注入上下文。当两者规则冲突时，Claude 可能**任意选择其一**（官方文档原文："If two rules contradict each other, Claude may pick one arbitrarily"）。

| 特性 | `CLAUDE.md` | `CLAUDE.local.md` |
|------|-----------|-------------------|
| 用途 | 团队共享的项目规则 | 个人的本地覆盖/偏好 |
| 提交到 Git | 是 | 否（应加入 `.gitignore`） |
| 加载顺序 | 先 | 后 |
| 冲突解决 | 不保证确定性 | 不保证确定性 |

由于冲突解决不确定，如需确保个人规则优先，建议在共享的 `CLAUDE.md` 中使用 `@import` 显式引入个人文件：

```markdown
# CLAUDE.md（团队共享）
团队通用规则...

@~/.claude/my-overrides.md
```

### 3.3 导入语法

```markdown
@path/to/file.md
@../shared/conventions.md
```

- 最大递归深度：**5 层**
- 相对路径从导入文件的位置解析

### 3.4 路径范围规则 (.claude/rules/)

#### 为什么需要 Rules

官方建议每个 CLAUDE.md 文件控制在 **200 行以下**。较长的文件消耗更多上下文并降低遵守度。当指令变多时，应使用 `@import` 或 `.claude/rules/` 进行分割。

#### Rules 是什么

`.claude/rules/` 下的每个 `.md` 文件是一条规则，与 CLAUDE.md 同等注入上下文。区别在于：规则支持通过 `paths` 字段限定**仅在操作匹配文件时加载**，而非无条件注入。

#### 加载行为

| 规则类型 | 加载时机 | 适用场景 |
|---------|---------|---------|
| 有 `paths` 字段 | Claude 读取匹配文件时**按需加载** | 特定目录/文件类型的专属规则 |
| 无 `paths` 字段 | 会话启动时**始终加载** | 需要全局生效的规则 |

#### 示例

```markdown
---
paths:
  - "src/api/**/*.{ts,tsx}"
---

- API 端点必须使用 Zod schema 验证输入
```

规则可放在 `~/.claude/rules/`（用户级，所有项目）或 `.claude/rules/`（项目级）。文件递归发现，支持子目录组织。运行 `/memory` 可查看所有已加载的规则。

### 3.5 自动记忆 (MEMORY.md) 行为

`MEMORY.md` 是**项目级别**的记忆，存储在 `~/.claude/projects/<project-hash>/memory/MEMORY.md`。每个项目有独立的 MEMORY.md，不会跨项目共享。如需跨项目的全局记忆，应使用 `~/.claude/CLAUDE.md`（用户级 CLAUDE.md）。

前 200 行在每次会话启动时自动加载。超出部分**不会丢失**，完整保留在磁盘上，Claude 可在会话中用 Read 工具主动读取。200 行是软限制，不可配置，没有自动裁剪机制。

**设计意图**：`MEMORY.md` 作为目录索引保持简洁，详细内容拆分到独立主题文件（如 `debugging.md`、`patterns.md`）。需要每次会话都加载的长指令应放在 `CLAUDE.md` 中（无行数限制，完整加载）。

![每次新会话：记忆如何装进上下文](claude%20code%20images/每次新会话：记忆如何装进上下文.jpg)

### 3.6 压缩后重新注入

CLAUDE.md 在自动压缩后**始终重新注入**，不会丢失。Claude 会在 `/compact` 后从磁盘重新读取并注入到会话中。

---

## 四、Skills — 技能系统

### 4.1 是什么

Skills 是**可复用的指令集**，通过斜杠命令调用或由模型自动触发。定义在 `SKILL.md` 文件中。

### 4.2 文件位置

| 范围 | 路径 |
|------|------|
| 企业 | 通过托管策略部署 |
| 个人 | `~/.claude/skills/<name>/SKILL.md` |
| 项目 | `.claude/skills/<name>/SKILL.md` |
| 插件 | `<plugin-root>/skills/<name>/SKILL.md` |

### 4.3 SKILL.md 格式

```markdown
---
name: deploy
description: 部署当前项目到指定环境
user-invocable: true
disable-model-invocation: false
allowed-tools:
  - Bash
  - Read
  - Write
model: sonnet
effort: high
context: fork
---

# 部署指令

## 步骤
1. 运行测试套件
2. 构建生产包
3. 使用部署脚本部署
```

### 4.4 Frontmatter 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 技能名称（默认为目录名），也是 `/slash-command` 名称 |
| `description` | string | 简短描述（用于渐进式披露，告诉 Claude 何时使用） |
| `user-invocable` | boolean | 设为 `false` 则从 `/` 菜单隐藏（默认 `true`） |
| `disable-model-invocation` | boolean | 设为 `true` 则阻止 Claude 自动加载（默认 `false`） |
| `allowed-tools` | list | 限制技能可用的工具，同时自动批准这些工具 |
| `model` | string | 覆盖使用的模型（sonnet/haiku/opus） |
| `effort` | string | 思考努力程度：low / medium / high / max（`max` 仅限 Opus 4.6） |
| `context` | string | 设为 `fork` 可在独立子代理中运行 |
| `agent` | string | 当 `context: fork` 时使用的子代理类型 |
| `hooks` | object | 附加到该技能生命周期的钩子 |
| `argument-hint` | string | 自动补全提示文本，如 `[issue-number]` |

### 4.5 触发方式：用户调用 vs 模型自动触发

Skill 支持两种触发方式，由 frontmatter 中的两个布尔字段控制：

| 配置 | 用户 `/skill-name` 触发 | 模型自动触发 | 描述何时加载到上下文 |
|------|------------------------|------------|---------------------|
| 默认（都不设置） | 可以 | 可以 | 描述始终在上下文中，调用时加载完整内容 |
| `user-invocable: false` | 不可以 | 可以 | 描述始终在上下文中，调用时加载完整内容 |
| `disable-model-invocation: true` | 可以 | 不可以 | 描述不在上下文中，用户调用时加载 |

- **`user-invocable`** — 控制用户能否通过 `/skill-name` 手动调用（默认 `true`）
- **`disable-model-invocation`** — 控制模型能否根据上下文自动判断并触发（默认 `false`，即默认开启自动触发）

默认情况下两者都是开启的，即**用户可以手动触发，模型也可以自动触发**。

### 4.6 渐进式披露 (Progressive Disclosure)

这是 Skill 系统的核心设计：

![上下文窗口管理与渐进式披露](claude%20code%20images/上下文窗口管理与渐进式披露.jpg)

### 4.7 动态上下文注入

支持在发送给 Claude 之前执行 Shell 命令：

```markdown
当前 PR diff：
!`gh pr diff`

当前分支：
!`git branch --show-current`
```

### 4.8 字符串替换

| 变量 | 展开结果 |
|------|---------|
| `$ARGUMENTS` | 传入技能的所有参数 |
| `$ARGUMENTS[N]` | 第 N 个参数（0 索引） |
| `$N` | `$ARGUMENTS[N]` 的简写，如 `$0`、`$1` |
| `${CLAUDE_SESSION_ID}` | 当前会话 ID |
| `${CLAUDE_SKILL_DIR}` | 技能 SKILL.md 所在目录的绝对路径 |

---

## 五、MCP Servers — 模型上下文协议

### 5.1 是什么

**MCP (Model Context Protocol)** 是 AI 工具集成的开放标准，定义了 AI 模型如何发现、加载和调用外部服务器提供的工具。

### 5.2 传输类型

| 类型 | 适用场景 | 状态 |
|------|---------|------|
| **stdio** | 本地进程（通过 stdin/stdout 通信） | 推荐 |
| **HTTP** | 远程 HTTP 服务器 | 推荐 |
| **SSE** | 远程 SSE 服务器 | 已废弃，建议改用 HTTP |

### 5.3 配置方式

```bash
# stdio 传输
claude mcp add [options] <name> -- <command> [args...]

# HTTP 传输
claude mcp add --transport http <name> <url>

# 所有选项（--transport, --env, --scope, --header）必须放在服务器名称之前
# -- 分隔服务器名称与传递给 MCP 服务器的命令和参数
```

### 5.4 项目 .mcp.json 格式

```json
{
  "mcpServers": {
    "playwright": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest"],
      "env": { "BROWSER": "chromium" }
    },
    "github": {
      "type": "http",
      "url": "https://api.github.com/mcp",
      "headers": { "Authorization": "Bearer ${GITHUB_TOKEN}" }
    }
  }
}
```

支持环境变量展开：`${VAR}` 和 `${VAR:-default}`，可用于 `command`、`args`、`env`、`url`、`headers` 字段。

### 5.5 配置范围

| 范围 | 文件 | 可见性 |
|------|------|--------|
| 本地 | `~/.claude.json`（按项目路径存储） | 仅当前项目，私有 |
| 项目 | `.mcp.json`（仓库根目录） | 团队共享，可提交到 Git |
| 用户 | `~/.claude.json`（全局存储） | 所有项目 |

### 5.6 工具命名

MCP 工具的命名规则：`mcp__<服务器名>__<工具名>`

例如：服务器 `memory` 上的 `search` 工具 → `mcp__memory__search`

### 5.7 工具搜索 / 懒加载

当 MCP 工具描述总量超过阈值（默认上下文窗口的 10%）时，启用**工具搜索**：

- 仅初始加载阈值内的工具描述
- 其他工具通过 `tool_search` 调用按需发现
- 通过环境变量控制：

| 设置值 | 行为 |
|--------|------|
| （未设置） | 默认启用；当 `ANTHROPIC_BASE_URL` 为非官方主机时禁用 |
| `true` | 始终启用 |
| `auto` | 当 MCP 工具超过上下文 10% 时启用 |
| `auto:N` | 自定义阈值，如 `auto:5` 表示 5% |
| `false` | 禁用，所有 MCP 工具 upfront 加载 |

> **注意**：工具搜索需要支持 `tool_reference` 块的模型（Sonnet 4 及以上、Opus 4 及以上）。Haiku 模型不支持工具搜索，此时所有 MCP 工具会 upfront 加载。

---

## 六、Hooks — 钩子系统

### 6.1 是什么

Hooks 是用户定义的脚本或命令，在 Claude Code 运行的**特定生命周期节点**自动执行。是唯一的**确定性、非 AI** 机制。

### 6.2 Hook 事件（完整列表）

#### 会话生命周期

| 事件 | 触发时机 | 能否阻塞？ |
|------|---------|-----------|
| `SessionStart` | 会话启动时 | 是 |
| `InstructionsLoaded` | 指令文件加载完成时 | 是 |
| `SessionEnd` | 会话结束时 | 是 |

#### 用户交互

| 事件 | 触发时机 | 能否阻塞？ |
|------|---------|-----------|
| `UserPromptSubmit` | 用户提交提示词时 | 是 |
| `Elicitation` | Claude 需要用户输入时 | 是 |
| `ElicitationResult` | 用户响应输入请求时 | 是 |

#### 工具调用

| 事件 | 触发时机 | 能否阻塞？ |
|------|---------|-----------|
| `PreToolUse` | 工具执行**之前** | 是 |
| `PostToolUse` | 工具执行**之后** | 是 |
| `PostToolUseFailure` | 工具执行失败后 | 是 |
| `PermissionRequest` | 权限请求时 | 是 |

#### 代理生命周期

| 事件 | 触发时机 | 能否阻塞？ |
|------|---------|-----------|
| `SubagentStart` | 子代理启动时 | 是 |
| `SubagentStop` | 子代理停止时 | 是 |

#### 任务控制

| 事件 | 触发时机 | 能否阻塞？ |
|------|---------|-----------|
| `Stop` | Claude 停止生成时 | 是 |
| `StopFailure` | 停止失败时 | 是 |
| `TaskCompleted` | 任务完成时 | 是 |
| `TeammateIdle` | 团队成员空闲时 | 是 |
| `Notification` | Claude 发送通知时 | 是 |

#### 上下文管理

| 事件 | 触发时机 | 能否阻塞？ |
|------|---------|-----------|
| `PreCompact` | 上下文压缩之前 | 是 |
| `PostCompact` | 上下文压缩之后 | 是 |

#### 配置与工作区

| 事件 | 触发时机 | 能否阻塞？ |
|------|---------|-----------|
| `ConfigChange` | 配置变更时 | 是 |
| `WorktreeCreate` | 工作树创建时 | 是 |
| `WorktreeRemove` | 工作树删除时 | 是 |

### 6.3 Hook 类型

| 类型 | AI 参与？ | 说明 |
|------|----------|------|
| `command` | 否 | 执行 Shell 命令，确定性执行 |
| `http` | 否 | POST 事件数据到 URL |
| `prompt` | 是 | 单轮 LLM 评估（默认使用 Haiku 模型），返回结构化 JSON 决策 |
| `agent` | 是 | 多轮 AI 验证，子代理可读取文件、搜索代码、检查代码库 |

### 6.4 退出码语义

| 退出码 | 含义 |
|--------|------|
| `0` | 成功 — 正常继续；Claude Code 解析 stdout 中的 JSON 输出（`UserPromptSubmit` 和 `SessionStart` 的 stdout 会作为上下文传递给 Claude） |
| `2` | **阻塞** — 停止当前操作；忽略 stdout。stderr 反馈方式因事件而异（`PreToolUse`/`Stop` 等会传递给 Claude；`Notification`/`SessionStart` 等仅显示给用户） |
| 其他 | 非阻塞 — 仅在详细模式（Ctrl+O）中显示 stderr，执行继续 |

### 6.5 PreToolUse 决策控制

PreToolUse hook 可通过 stdout 输出 JSON 来控制行为：

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "permissionDecisionReason": "已通过安全检查",
    "updatedInput": { "command": "echo 'safe alternative'" },
    "additionalContext": "当前环境：开发"
  }
}
```

| 字段 | 说明 |
|------|------|
| `hookEventName` | 必须为事件名称（如 `"PreToolUse"`），Claude Code 据此选择对应的 schema 解析 |
| `permissionDecision` | `"allow"`（绕过权限）/ `"deny"`（阻止工具调用）/ `"ask"`（提示用户确认） |
| `permissionDecisionReason` | 人类可读的解释（allow/ask 时显示给用户，deny 时反馈给 Claude） |
| `updatedInput` | 修改工具的输入参数 |
| `additionalContext` | 追加上下文供 Claude 参考 |

### 6.6 配置示例

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": ".claude/hooks/block-rm.sh" }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [{ "type": "command", "command": "prettier --write $FILEPATH" }]
      }
    ],
    "Stop": [
      {
        "hooks": [{ "type": "command", "command": "git diff --stat" }]
      }
    ]
  }
}
```

`matcher` 字段为可选的正则表达式。省略、使用 `""` 或 `"*"` 表示匹配所有。部分事件不支持 matcher，添加了会被**静默忽略**：`UserPromptSubmit`、`Stop`、`TeammateIdle`、`TaskCompleted`、`WorktreeCreate`、`WorktreeRemove`。

支持 matcher 的事件及匹配对象：`PreToolUse`/`PostToolUse`/`PostToolUseFailure`/`PermissionRequest`（工具名）、`SessionStart`（启动方式）、`SubagentStart`/`SubagentStop`（agent 类型）、`Notification`（通知类型）、`ConfigChange`（配置来源）等。

### 6.7 异步 Hook

```json
{
  "type": "command",
  "command": "notify-send 'Claude is working'",
  "async": true
}
```

- 仅 `type: "command"` 支持异步
- 异步 Hook 无法阻塞或控制 Claude 行为
- 后台进程结束后，如果产生包含 `systemMessage` 或 `additionalContext` 的 JSON 响应，会在下一轮对话中传递给 Claude

### 6.8 在 Skill 和 Agent 中定义 Hooks

Hooks 可以在 Skill 和 Agent 的 frontmatter 中定义，仅在该组件的生命周期内生效：

```yaml
# Skill frontmatter 示例
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/security-check.sh"
```

> **注意**：在 Agent 中定义的 `Stop` hook 会自动转换为 `SubagentStop` 事件，因为子代理完成时触发的是 `SubagentStop` 而非 `Stop`。

---

## 七、Subagents — 子代理

### 7.1 是什么

Subagents 是在**独立上下文窗口**中运行的 Claude Code 实例，拥有自定义系统提示词、特定工具访问和独立权限。用于复杂的多步骤任务。

### 7.2 内置 Agent

| Agent | 模型 | 工具 | 用途 |
|-------|------|------|------|
| **Explore** | Haiku（快速低延迟） | 只读（禁止 Write 和 Edit） | 快速代码探索 |
| **Plan** | 继承主对话 | 只读 | 架构规划 |
| **General-purpose** | 继承主对话 | 全部 | 复杂多步骤任务 |

### 7.3 自定义 Agent

定义在 `.claude/agents/<name>.md`（项目）或 `~/.claude/agents/<name>.md`（用户）：

```markdown
---
name: security-reviewer
description: 执行代码安全审查
tools:
  - Read
  - Grep
  - Glob
model: sonnet
permissionMode: dontAsk
maxTurns: 50
memory: project
isolation: worktree
---

你是一个安全审查员，分析代码变更中的安全漏洞...
```

### 7.4 Agent Frontmatter 字段

| 字段 | 必需 | 说明 |
|------|------|------|
| `name` | 是 | Agent 标识符 |
| `description` | 是 | 渐进式披露的描述 |
| `tools` | 否 | 允许的工具列表 |
| `disallowedTools` | 否 | 禁止的工具列表 |
| `model` | 否 | 模型覆盖 |
| `permissionMode` | 否 | 权限模式（default / acceptEdits / dontAsk / bypassPermissions / plan；不支持 auto） |
| `maxTurns` | 否 | 最大循环迭代次数 |
| `skills` | 否 | 可用的 Skills |
| `mcpServers` | 否 | 可用的 MCP 服务器（可限定范围） |
| `hooks` | 否 | Agent 专属 Hooks |
| `memory` | 否 | 持久记忆（user/project/local） |
| `background` | 否 | 后台运行 |
| `effort` | 否 | 思考努力程度 |
| `isolation` | 否 | `worktree` = git worktree 隔离 |

### 7.5 关键约束

- 子代理在**独立上下文窗口**中运行，不与父会话共享上下文
- 子代理支持独立的自动压缩
- 子代理**不能再生成子代理**（不允许嵌套）
- 子代理接收父会话的摘要以理解任务
- 如果需要嵌套委派，应使用 Skills 或从主对话链式调用子代理

---

## 八、Plugins — 插件系统

### 8.1 是什么

Plugins 是 Skills、Agents、Hooks、MCP Servers 和 Settings 的**打包分发机制**，提供命名空间防止冲突。

### 8.2 目录结构

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json         # 清单文件（可选）
├── skills/                  # 推荐：新技能放这里
│   ├── hello/
│   │   └── SKILL.md
│   └── deploy/
│       └── SKILL.md
├── commands/                # 遗留：旧版命令目录，仍可用但不推荐
│   └── legacy-cmd.md
├── agents/
│   └── reviewer.md
├── hooks/
│   └── hooks.json
├── .mcp.json
├── .lsp.json
└── settings.json
```

> **注意**：`commands/` 目录为遗留目录，仍向后兼容。新插件应使用 `skills/` 目录。两者不能同时包含同名技能。

### 8.3 plugin.json 清单

清单文件是**可选的**。如果省略，Claude Code 会自动发现默认位置的组件，并从目录名推导插件名。如果提供，则 `name` 字段是**唯一必需的字段**。

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "部署和审查工作流插件",
  "author": {
    "name": "Your Name",
    "email": "you@example.com",
    "url": "https://example.com"
  },
  "homepage": "https://github.com/you/my-plugin",
  "repository": "https://github.com/you/my-plugin",
  "license": "MIT",
  "keywords": ["deploy", "review"]
}
```

### 8.4 命名空间

插件内的技能以 `/plugin-name:skill-name` 调用，防止多个插件的同名技能冲突：

```
/my-plugin:deploy      ← 调用 my-plugin 插件的 deploy 技能
/my-plugin:review      ← 调用 my-plugin 插件的 review 技能
```

### 8.5 安装管理

```bash
# 安装（scope 可选：user/project/local，默认 user）
claude plugin install <name> [-s scope]

# 卸载（也可用 remove 或 rm）
claude plugin uninstall <name>

# 本地测试
claude --plugin-dir ./my-plugin

# 启用/禁用
claude plugin enable <name>
claude plugin disable <name>

# 更新
claude plugin update <name>
```

### 8.6 插件环境变量

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write",
      "hooks": [{
        "type": "command",
        "command": "${CLAUDE_PLUGIN_ROOT}/scripts/format.sh"
      }]
    }]
  }
}
```

`${CLAUDE_PLUGIN_ROOT}` 包含插件目录的绝对路径，用于确保无论安装位置如何都能正确引用路径。

### 8.8 安装范围

| 范围 | 存储位置 | 用途 |
|------|---------|------|
| `user` | `~/.claude/settings.json` | 个人全局插件（默认） |
| `project` | `.claude/settings.json` | 团队共享，提交到 Git |
| `local` | `.claude/settings.local.json` | 项目专属，gitignored |
| `managed` | 托管策略 | 组织级部署，只读，仅可更新 |

> `plugin install` 仅支持 `user`/`project`/`local` 三种范围。`managed` 由管理员通过托管策略部署，用户无法直接安装。

### 8.7 插件缓存

通过市场安装的插件会被复制到 `~/.claude/plugins/cache/`（而非原地使用）。通过 `--plugin-dir` 加载的插件则在原地使用。

---

## 九、协调工作流

### 9.1 任务执行完整流程

![任务执行完整流程](claude%20code%20images/任务执行完整流程.jpg)

### 9.2 六大机制协作关系

![六大机制协作关系](claude%20code%20images/六大机制协作关系.jpg)

---

## 十、文件路径速查表

| 组件 | 用户范围 | 项目范围 |
|------|---------|---------|
| Settings | `~/.claude/settings.json` | `.claude/settings.json` |
| Local Settings | — | `.claude/settings.local.json` |
| Skills | `~/.claude/skills/<name>/SKILL.md` | `.claude/skills/<name>/SKILL.md` |
| Agents | `~/.claude/agents/<name>.md` | `.claude/agents/<name>.md` |
| MCP Config | `~/.claude.json`（全局或按项目路径） | `.mcp.json` |
| CLAUDE.md | `~/.claude/CLAUDE.md` | `CLAUDE.md` 或 `.claude/CLAUDE.md` |
| 本地指令 | — | `CLAUDE.local.md` |
| Rules | — | `.claude/rules/<name>.md` |
| Auto Memory | `~/.claude/projects/<hash>/memory/MEMORY.md` | — |
| Plugin Cache | `~/.claude/plugins/cache/`（市场插件） | — |
| Hooks | 定义在 settings.json 或 plugin hooks/hooks.json | 定义在 settings.json 或 plugin hooks/hooks.json |

![配置文件位置速查：用户级 vs 项目级](claude%20code%20images/配置文件位置速查：用户级%20vs%20项目级.jpg)

---

## 十一、核心对比：何时用什么

| 场景 | 推荐机制 | 原因 |
|------|---------|------|
| 团队编码规范 | CLAUDE.md | 每次会话自动加载，始终生效 |
| 可复用的工作流（部署、审查） | Skill | 按需调用，渐进式披露省上下文 |
| 需要外部 API/数据 | MCP Server | 协议标准化，懒加载 |
| 文件编辑后自动格式化 | Hook (PostToolUse) | 确定性执行，不依赖 AI 判断 |
| 阻止危险命令 | Hook (PreToolUse) | 可阻塞，强制执行 |
| 大型代码审查任务 | Subagent | 独立上下文，不污染主会话 |
| 跨项目分享整套配置 | Plugin | 打包分发，命名空间隔离 |
| 特定目录的代码规则 | `.claude/rules/` | 路径范围限定 |
| 跨会话记住用户偏好 | Auto Memory | Claude 自动维护 |

![何时用什么？决策速查](claude%20code%20images/何时用什么？决策速查.jpg)
