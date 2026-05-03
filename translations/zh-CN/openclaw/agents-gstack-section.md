## 编码任务 (gstack)

### 规则(不可协商)

1. **始终生成会话,绝不重定向。** 当用户要求使用任何 gstack 技能时,
   始终通过 sessions_spawn 生成 Claude Code 会话。绝不告诉用户自己打开
   Claude Code。绝不说"这需要在 Claude Code 中运行"。
   绝不说"你需要为此打开 Claude Code"。直接执行即可。

2. **解析仓库。** 如果用户指定了仓库或项目,将工作目录设置为该仓库路径。
   如果仓库路径未知,询问是哪个仓库——不要推脱说让用户打开 Claude Code。

3. **Autoplan 端到端运行。** 特别是对于 /autoplan:生成会话,
   让它运行完整的审查流程(CEO → 设计 → 工程),完成后,
   在聊天中报告计划。将计划写入内存,以便用户稍后查找。
   用户永远不需要离开 Telegram。

### 调度路由

当被要求进行编码工作时,选择调度层级:

**SIMPLE:** "修复这个拼写错误","更新那个配置",单文件更改
→ sessions_spawn(runtime: "acp", prompt: "<仅任务内容>")

**MEDIUM:** 多文件功能、重构、技能编辑
→ sessions_spawn(runtime: "acp", prompt: "<gstack-lite 内容>\n\n<任务>")

**HEAVY:** 需要特定的 gstack 方法论
→ sessions_spawn(runtime: "acp", prompt: "Load gstack. Run /qa https://...")
  技能:/cso、/review、/qa、/ship、/investigate、/design-review、/benchmark、/gstack-upgrade

**FULL:** 构建完整功能,多天范围,需要规划 + 审查
→ sessions_spawn(runtime: "acp", prompt: "<gstack-full 内容>\n\n<任务>")
  Claude Code 运行:/autoplan → 实现 → /ship → 报告结果

**PLAN:** 用户想要规划 Claude Code 项目、制定功能规格或在编写任何代码之前进行设计
→ sessions_spawn(runtime: "acp", prompt: "<gstack-plan 内容>\n\n<任务>")
  Claude Code 运行:/office-hours → /autoplan → 保存计划文件 → 报告结果
  将计划链接持久化到内存/知识存储。
  当用户准备实现时,生成一个指向该计划的新 FULL 会话。

### 决策启发式

- 能在 <10 行代码内完成吗?→ **SIMPLE**
- 涉及多个文件但方法显而易见吗?→ **MEDIUM**
- 用户是否指定了特定技能(/cso、/review、/qa)?→ **HEAVY**
- "升级 gstack"、"更新 gstack" → **HEAVY** 并使用 `Run /gstack-upgrade`
- 是功能、项目还是目标(而非任务)?→ **FULL**
- 用户是否想要规划某事而不立即实现?→ **PLAN**