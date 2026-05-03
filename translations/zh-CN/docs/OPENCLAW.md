# gstack x OpenClaw 集成

gstack 作为方法论来源与 OpenClaw 集成,而非移植代码库。
OpenClaw 的 ACP 运行时原生生成 Claude Code 会话。gstack 提供
规划纪律和方法论,使这些会话更加高效。

这是一个编码为提示文本的轻量级协议。无守护进程。无 JSON-RPC。
无兼容性矩阵。提示就是桥梁。

## 架构

```
  OpenClaw                               gstack repo
  ─────────────────────                    ──────────────
  Orchestrator: 消息传递,                  方法论 + 规划的
  日历, 内存, EA                           真实来源
       │                                        │
       ├── 原生技能 (对话式)                     ├── 通过 gen-skill-docs 管道
       │   office-hours, ceo-review,            │   生成原生技能
       │   investigate, retro                   │
       │                                        ├── 生成 gstack-lite
       ├── sessions_spawn(runtime: "acp")       │   (规划纪律)
       │       │                                │
       │       └── Claude Code                  ├── 生成 gstack-full
       │           └── gstack 安装在             │   (完整管道)
       │               ~/.claude/skills/gstack  │
       │                                        └── docs/OPENCLAW.md (本文件)
       └── 调度路由 (AGENTS.md)
```

## 调度路由

OpenClaw 在生成时决定使用哪个层级的 gstack 支持:

| 层级 | 使用场景 | 提示前缀 |
|------|------|---------------|
| **Simple** | 单文件编辑、拼写错误、配置更改 | 不注入 gstack 上下文 |
| **Medium** | 多文件功能、重构 | 附加 gstack-lite CLAUDE.md |
| **Heavy** | 需要特定 gstack 技能 | "Load gstack. Run /X" |
| **Full** | 完整功能、目标、项目 | 附加 gstack-full 管道 |
| **Plan** | "帮我规划一个 Claude Code 项目" | 附加 gstack-plan 管道 |

### 决策启发式

- 能在 <10 行代码内完成吗? -> **Simple**
- 涉及多个文件但方法显而易见吗? -> **Medium**
- 用户指定了特定技能 (/cso, /review, /qa) 吗? -> **Heavy**
- 是功能、项目或目标(而非任务)吗? -> **Full**
- 用户想要为 Claude Code 规划某事但尚未实现吗? -> **Plan**

### 调度路由指南 (用于 AGENTS.md)

完整的即用粘贴部分位于 `openclaw/agents-gstack-section.md`。
将其复制到你的 OpenClaw AGENTS.md 中。

关键行为规则(这些规则位于调度层级之上):

1. **始终生成,永不重定向。** 当用户要求使用任何 gstack 技能时,
   始终生成 Claude Code 会话。永远不要告诉用户打开 Claude Code。
2. **解析仓库。** 如果用户指定了仓库,设置工作目录。如果
   未知,询问是哪个仓库。
3. **自动规划端到端运行。** 生成会话,让它运行完整管道,在聊天中
   报告结果。用户无需离开 Telegram。

### CLAUDE.md 冲突处理

在已有 CLAUDE.md 的仓库中生成 Claude Code 时,将
gstack-lite/full 作为新部分附加。不要替换仓库现有的指令。

## gstack 为 OpenClaw 生成的内容

所有产物位于 `openclaw/` 目录,由
`bun run gen:skill-docs --host openclaw` 生成:

### gstack-lite (Medium 层级)
`openclaw/gstack-lite-CLAUDE.md` — 约 15 行规划纪律:
1. 修改前阅读每个文件
2. 编写 5 行计划: 做什么、为什么、哪些文件、测试用例、风险
3. 使用决策原则解决歧义
4. 报告完成前自我审查
5. 完成报告: 交付内容、做出的决策、任何不确定的事项

A/B 测试: 2 倍时间,输出质量显著提升。

### gstack-full (Full 层级)
`openclaw/gstack-full-CLAUDE.md` — 链接现有 gstack 技能:
1. 阅读 CLAUDE.md 并理解项目
2. 运行 /autoplan (CEO + 工程 + 设计审查)
3. 实现批准的计划
4. 运行 /ship 创建 PR
5. 报告 PR URL 和决策

### gstack-plan (Plan 层级)
`openclaw/gstack-plan-CLAUDE.md` — 完整审查流程,无实现:
1. 运行 /office-hours 生成设计文档
2. 运行 /autoplan (CEO + 工程 + 设计 + DX 审查 + codex 对抗性审查)
3. 将审查后的计划保存到 `plans/<project-slug>-plan-<date>.md`
4. 报告: 计划路径、摘要、关键决策、推荐的下一步

编排器将计划链接持久化到其自己的内存存储(brain 仓库、
知识库或 AGENTS.md 中配置的任何内容)。当用户
准备构建时,生成引用已保存计划的 FULL 会话。

### 原生方法论技能
发布到 ClawHub。使用 `clawhub install` 安装:
- `gstack-openclaw-office-hours` — 产品质询(6 个强制性问题)
- `gstack-openclaw-ceo-review` — 战略挑战(10 部分审查,4 种模式)
- `gstack-openclaw-investigate` — 运营调试(4 阶段方法论)
- `gstack-openclaw-retro` — 运营回顾(每周审查)

源代码位于 gstack 仓库的 `openclaw/skills/` 中。这些是为 OpenClaw 
对话上下文手工改编的 gstack 方法论。
无 gstack 基础设施(无浏览、无遥测、无前言)。

## 生成会话检测

当 Claude Code 在 OpenClaw 生成的会话中运行时,应设置 `OPENCLAW_SESSION`
环境变量。gstack 检测到此变量并调整:
- 跳过交互式提示(自动选择推荐选项)
- 跳过升级检查和遥测提示
- 专注于任务完成和散文报告

在 sessions_spawn 中设置环境变量: `env: { OPENCLAW_SESSION: "1" }`

## 安装

对于 OpenClaw 用户: 告诉你的 OpenClaw 代理 "install gstack for openclaw."

代理应该:
1. 将 gstack-lite CLAUDE.md 安装到其编码会话模板中
2. 安装 4 个原生方法论技能
3. 将调度路由添加到 AGENTS.md
4. 通过测试生成进行验证

对于 gstack 开发者: `./setup --host openclaw` 输出此文档。
实际产物由 `bun run gen:skill-docs --host openclaw` 生成。

## 我们不做的事

- 无调度守护进程(ACP 处理会话生成)
- 无 Clawvisor 中继(不需要安全层)
- 无双向学习桥接(brain 仓库是知识存储)
- 无 JSON 模式或协议版本控制
- gstack 不提供 SOUL.md(OpenClaw 有自己的)
- 不完全移植技能(编码技能保持 Claude Code 原生)