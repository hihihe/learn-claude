# Claude Code 工作原理与扩展系统完整指南

> 基于 Claude Code 官方文档整理，配合精美架构图，帮助你快速理解 Claude Code 的核心机制。

## 📖 文档

- [claude-code-guide.md](./claude-code-guide.md) — 完整指南

## 🖼️ 架构图预览

| 三阶段 Agentic Loop | 启动流程 |
|:---:|:---:|
| ![Agentic Loop](./claude%20code%20images/Agentic%20Loop%20三阶段自主循环.jpg) | ![启动流程](./claude%20code%20images/Claude%20Code%20启动流程.jpg) |

| 六大扩展机制 | 六大机制协作 |
|:---:|:---:|
| ![六大扩展机制](./claude%20code%20images/六大扩展机制全景图.jpg) | ![协作关系](./claude%20code%20images/六大机制协作关系.jpg) |

## 📚 核心内容

### 一、整体架构
- 三阶段 Agentic Loop（收集上下文 → 执行动作 → 验证结果）
- 启动流程与配置加载优先级
- 上下文窗口管理策略
- 权限模式详解

### 二、六大扩展机制

| 机制 | 用途 | 类比 |
|------|------|------|
| **CLAUDE.md** | 始终生效的项目规则 | 团队手册 |
| **Skills** | 按需调用的指令集 | 操作手册 |
| **Subagents** | 独立上下文执行任务 | 专项员工 |
| **MCP Servers** | 外部工具桥接 | 外部工具箱 |
| **Hooks** | 生命周期自动触发 | 自动触发器 |
| **Plugins** | 打包分发以上所有 | 运输集装箱 |

### 三、关键文件路径速查

| 组件 | 项目范围 | 用户范围 |
|------|---------|---------|
| Settings | `.claude/settings.json` | `~/.claude/settings.json` |
| Skills | `.claude/skills/<name>/SKILL.md` | `~/.claude/skills/<name>/SKILL.md` |
| Agents | `.claude/agents/<name>.md` | `~/.claude/agents/<name>.md` |
| MCP Config | `.mcp.json` | `~/.claude.json` |
| 项目指令 | `CLAUDE.md` | `~/.claude/CLAUDE.md` |
| 自动记忆 | — | `~/.claude/projects/<hash>/memory/MEMORY.md` |

## 🔗 推荐资源

### 必读文章

| 文章 | 作者 | 说明 |
|------|------|------|
| [你不知道的 Claude Code：架构、治理与工程实践](https://x.com/HiTw93/status/2032091246588518683?s=20) | [@HiTw93](https://x.com/HiTw93) | 深度解析 Claude Code 架构 |
| [你不知道的 Agent：原理、架构与工程实践](https://x.com/HiTw93/status/2034627967926825175?s=20) | [@HiTw93](https://x.com/HiTw93) | Agent 原理与工程实践 |
| [10 个 Claude Code 隐藏命令](https://x.com/Khazix0918/status/2034842244600275340?s=20) | [@Khazix0918](https://x.com/Khazix0918) | /btw、/rewind、/insights 等实用命令 |
| [深度解析 OpenClaw 的爆火](https://x.com/Charles77xixi/status/2035366324981858743?s=20) | [@Charles77xixi](https://x.com/Charles77xixi) | Agent 发展趋势与核心概念解析 |
| [Agent Skills 设计哲学和实战进化](https://x.com/dotey/status/2036114136245969025?s=20) | [@dotey](https://x.com/dotey) | Skills 设计方法论与 baoyu-skills 开源项目 |

### 开源项目

| 项目 | 说明 |
|------|------|
| [garrytan/gstack](https://github.com/garrytan/gstack) | YC CEO Garry Tan 开源的 AI 工作流 Skill，3w+ Star |
| [jimliu/baoyu-skills](https://github.com/JimLiu/baoyu-skills) | 高质量 Skills 合集，10K+ Star，19W+ 安装 |
| [YouMind Skills](https://youmind.com/skills) | AI Skills 发现平台，含大量高质量 Claude Code 可用 Skill |

### 播客访谈

- [张小珺 × 谢赛宁 7 小时马拉松访谈](https://www.xiaoyuzhoufm.com/episode/69b77577f8b8079bfa8eb837)

### 技术分享

- [Thariq: 构建 Claude Code 的教训——我们如何使用技能](https://x.com/trq212/status/2033949937936085378?s=20)

---

## 📄 License

MIT
