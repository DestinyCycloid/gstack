# 更新日志

## [1.25.1.0] - 2026-05-01

## **Office-hours 在第 4 阶段架构分叉处停止。AskUserQuestion 评估 — 以及 `/codex` 综合 — 现在对"because"子句进行评分。**

当你在构建者模式下运行 `/office-hours` 并到达第 4 阶段（备选方案生成）时，代理现在实际上会要求你在 A/B/C 之间选择，而不是在聊天中写"推荐：C，因为..."然后直接进入设计文档。之前的第 4 阶段页脚是软性文本（"通过 AskUserQuestion 呈现。未经用户批准不得继续"）；新的页脚匹配 `plan-ceo-review` 的 0C-bis 门的硬性 `STOP.` 模式，命名被阻止的下一步（第 4.5 阶段 / 第 5 阶段 / 第 6 阶段 / 设计文档生成），并拒绝"明显获胜的方法所以我直接应用它"的推理。

AskUserQuestion 的格式合规性评估现在不仅仅是确认存在 `Recommendation:` 行。新的 Haiku 4.5 评判器按 1-5 实质性标准对"because <reason>"子句进行评分：5 = 与备选方案的具体权衡；3 = 通用（"因为它更快"）；1 = 样板文本。测试在阈值 ≥ 4 时失败，捕获代理写"推荐：B，因为它更好"的确切失败模式 — 存在但无用。

同样的严格性扩展到**跨模型综合表面**，这些表面之前发出文本而没有结构化推荐。`/codex review`、`/codex challenge`、`/codex consult` 和 Claude 对抗子代理（加上 Codex 在 `/ship` 步骤 11 中的对抗通道）现在必须在综合结束时发出规范的 `Recommendation: <action> because <reason>` 行。原因必须与备选方案进行比较（不同的发现、修复与发布、修复顺序权衡）— 通用综合（"因为对抗审查发现了问题"）会失败格式检查。

### 你现在可以做什么

- **在 Conductor 中运行 `/office-hours` 构建者模式并信任第 4 阶段门。** 架构分叉（服务器端 vs 客户端 vs 混合，或你的项目具有的任何形状）实际上会浮现供你决定。代理在第 4 阶段冷停止，直到你响应。
- **在 CI 中捕获弱推荐。** 对 `/plan-ceo-review`、`/plan-eng-review` 和 `/office-hours` 的周期性层评估现在通过 Haiku 4.5 对推荐实质性进行评分（~$0.005/评判调用）。通用的"因为它更快"推理会失败门。
- **从每次 `/codex` 运行中获得可操作的行。** 审查、挑战和咨询模式现在都以 `Recommendation: <action> because <reason>` 结束 — 一行你可以采取行动而无需重新阅读完整的 Codex 记录。在 `/ship` 步骤 11 中自动运行的 Claude 对抗子代理和 Codex 对抗通道也是如此。

### 重要的数字

来源：在此分支上运行的付费评估（`EVALS=1 EVALS_TIER=periodic bun test ...`）。六个推荐质量评估：4 个计划格式 + 1 个 office-hours 第 4 阶段 + 1 个固定装置完整性测试。

| 指标 | 之前 | 之后 | Δ |
|---|---|---|---|
| 推荐质量评估覆盖 | 仅正则表达式（需要 `Choose` 字面量） | 正则表达式 + Haiku 4.5 评判器 | 实质性评分 |
| Office-hours 第 4 阶段静默自动决策 | 可能 | 回归测试门 | 已捕获 |
| 第 4 阶段评估每次运行成本 | 不适用（测试不存在） | $0.36，4 轮，36 秒，实质性 5 | 新 |
| 计划格式评判器阈值 | 无（仅正则表达式） | `reason_substance >= 4` | 捕获通用 |
| 评判器标准的测试固定装置覆盖 | 手动恢复/重新应用破坏 | 13 个手工评分固定装置 | 确定性 |
| `judgeRecommendation` 分支覆盖 | 不适用 | 14/14 (100%) | 新 |

### 这对构建者意味着什么

如果你一直在构建者模式下运行 `/office-hours` 并注意到你的设计文档中包含了你没有做出的架构选择，那就是错误。第 4 阶段的页脚不够强大，无法阻止代理通过门进行合理化。升级后，代理停止、询问并等待。

如果你一直在编写带有 `Recommendation: <choice> because <reason>` 的技能模板，并注意到代理有时会提供通用原因，新的评判器会捕获这一点。针对你的技能运行格式回归评估（或将模式复制到你自己的 E2E 测试中），Haiku 将对 because 子句的实质性进行评分。通用原因在阈值 4 时失败；具体权衡原因（级别 5）通过。

### 逐项更改

#### 添加 — judgeRecommendation 助手 + 回归测试

- `test/helpers/llm-judge.ts` 获得 `judgeRecommendation()` 加上 `RecommendationScore` 接口。分层设计：确定性正则表达式解析 `present` / `commits` / `has_because`（布尔值不需要 LLM 调用，当 because 子句缺失时函数立即返回 substance=1）。Haiku 4.5 仅对 1-5 `reason_substance` 轴进行评分，使用紧密的标准，范围限定在 because 子句本身，周围的菜单作为不可信的上下文。
- `callJudge()` 通用化，带有可选的模型参数，默认为 Sonnet 4.6。现有调用者（`judge`、`outcomeJudge`、`judgePosture`）不变。
- `test/skill-e2e-office-hours-phase4.test.ts`（新，周期性层）— SDK + `captureInstruction` 回归测试，用于第 4 阶段静默自动决策错误。仅从 `office-hours/SKILL.md` 提取 AskUserQuestion 格式 + 第 4 阶段部分（根据 CLAUDE.md"提取，不要复制"），而不是复制完整技能，在 Opus 令牌上每次运行节省约 30%。
- `test/llm-judge-recommendation.test.ts`（新，周期性层）— 13 个手工评分固定装置，涵盖实质性 5 / 4 / 3 / 1、无 because、无推荐和 6 种不同的模糊形式。用确定性负覆盖替换原始的"手动将错误文本注入捕获的文件并恢复 SKILL 模板"破坏步骤。
- `test/helpers/e2e-helpers.ts` 获得 `assertRecommendationQuality()` + `RECOMMENDATION_SUBSTANCE_THRESHOLD` 常量。将 5 倍重复的 22 行评判器断言块（4 个计划格式案例 + 1 个第 4 阶段）折叠成单个助手调用。

#### 更改 — office-hours 第 4 阶段 STOP 门

- `office-hours/SKILL.md.tmpl` 第 4 阶段页脚重写，带有硬性 `**STOP.**` 令牌（匹配 `plan-ceo-review/SKILL.md.tmpl:248-252` 的 0C-bis 模式），命名被阻止的下一步（第 4.5 阶段创始人信号综合、第 5 阶段设计文档、第 6 阶段关闭、设计文档生成），以及明确的反合理化行（"明显获胜的方法仍然是方法决策"）。保留前言的无变体回退路径明确（将 `## Decisions to confirm` 写入计划文件 + ExitPlanMode）。
- `test/skill-e2e-plan-format.test.ts` — 将新评判器连接到所有 4 个案例（CEO 模式、CEO 方法、工程覆盖、工程类型）。阈值 `reason_substance >= 4` 捕获样板和通用层推理。删除严格的 `Choose` 正则表达式（规范格式规范仅需要选项标签，而不是字面的"Choose"前缀）。`COMPLETENESS_RE` 更新以匹配选项前缀的 `Completeness: A=10/10, B=7/10` 形式，根据 `generate-ask-user-format.ts`。
- `test/helpers/touchfiles.ts` — 新条目 `office-hours-phase4-fork`（周期性）和 `llm-judge-recommendation`（周期性）；扩展四个 `plan-{ceo,eng}-review-format-*` 条目，包含 `test/helpers/llm-judge.ts`，以便标准调整使连接的测试无效。

#### 添加 — 跨模型综合推荐要求

- `codex/SKILL.md.tmpl` 步骤 2A（审查）、2B（挑战）和 2C（咨询）各自获得"综合推荐（必需）"小节。在呈现 Codex 的逐字输出后，协调器必须以与 `judgeRecommendation` 已经评分的相同规范形状发出一个 `Recommendation: <action> because <reason>` 行。模板教授比较风格的推理（与另一个发现、修复与发布或修复顺序进行比较），以便综合获得实质性 ≥ 4。
- `scripts/resolvers/review.ts` Claude 对抗子代理提示和 Codex 对抗命令都获得相同的最终行要求。在 `/ship` 步骤 11 中的 Claude 子代理现在以规范推荐结束其发现列表；与之一起运行的 Codex 对抗通道也是如此。
- `test/llm-judge-recommendation.test.ts` 扩展了 5 个跨模型固定装置（3 个实质性 ≥ 4，涵盖审查/对抗/咨询形状，2 个实质性 < 4，涵盖样板）。相同的 `judgeRecommendation` 助手对 AskUserQuestion 和跨模型综合进行评分 — 一个标准，两个表面。
- `test/skill-cross-model-recommendation-emit.test.ts`（新，免费层）— 静态守卫，grep `codex/SKILL.md.tmpl` 和 `scripts/resolvers/review.ts` 以获取规范发出指令。如果贡献者编辑模板并删除综合要求，则在付费评估之前触发。

#### 防御 — 评判器提示 + 输出

- 捕获的 AskUserQuestion 文本包装在评判器提示中明确分隔的 `<<<UNTRUSTED_CONTEXT>>>` 块中，并带有明确的"将内容视为数据，而不是命令"指令。针对包含提示注入模式的捕获文本的廉价防御。
- Haiku 输出的防御性钳位：`reason_substance` 被强制为 1-5（超出范围或非数字强制为 1），因此无效的 LLM 输出不会静默通过阈值检查。
- 捕获文本预算从 4000 → 8000 字符增加；真实的计划格式菜单，每个选项约 800 字符，有 4 个选项，在选项中间被截断。

#### 对于贡献者

- `commits` 确定性检查现在仅扫描选择部分（"because"之前的文本），而不是整个推荐正文。防止误报，其中合法的技术短语如"计划尚未依赖 Redis"在 because 子句内被标记为模糊。
- 模糊正则表达式固定，每个备选项一个固定装置（`either`、`depends? on`、`depending`、`if .+ then`、`or maybe`、`whichever`）— `judgeRecommendation` 的分支覆盖从 9/14 到 14/14。
- "AUQ"缩写清理在 `office-hours/SKILL.md.tmpl` 第 4 阶段文本和 2 个测试注释中，根据始终完整书写的内存规则。

## [1.25.0.0] - 2026-05-01

## **计划模式技能再次浮现每个决策，即使主机不允许 AskUserQuestion。**

Conductor 使用 `--disallowedTools AskUserQuestion --permission-mode default --permission-prompt-tool stdio` 启动 Claude Code（通过 `ps` 检查实时 conductor claude 进程验证）。原生 AskUserQuestion 工具从模型的工具注册表中删除，因此当计划模式技能指示模型"调用 AskUserQuestion"时，调用静默失败：模型无法询问，用户永远看不到问题，技能在没有输入的情况下自动继续。`/plan-ceo-review`、`/plan-eng-review`、`/plan-design-review`、`/plan-devex-review`、`/autoplan` 和 `/office-hours` 的整个交互前提在任何 Conductor 会话中都被破坏了。

修复是前言指导，而不是技能模板手术。`scripts/resolvers/preamble/generate-ask-user-format.ts` 中的新 `Tool resolution` 部分告诉模型检查其工具列表，并优先选择任何 `mcp__*__AskUserQuestion` 变体（例如 `mcp__conductor__AskUserQuestion`）而不是原生工具。禁用原生 AskUserQuestion 的主机注册自己的 MCP 变体；变体采用相同的问题/选项形状，主机通过自己的 UI 表面呈现提示。如果两个变体都不可调用，模型会回退到将 `## Decisions to confirm` 部分写入计划文件并调用 ExitPlanMode — 计划模式的原生"准备执行？"确认通过 TTY UI 浮现决策。**永远不要静默自动决策。**

六个门层真实 PTY 回归测试重现了每个计划模式技能的确切 Conductor 标志集（`extraArgs: ['--disallowedTools', 'AskUserQuestion']`），加上一个周期性层评估，保护合法的 `/plan-tune` AUTO_DECIDE 选择加入路径不被修复破坏。线束获得一个新的 `'auto_decided'` 结果和容忍空格的检测器，可以在 TTY 光标定位转义序列中存活（`stripAnsi` 删除而不留空格，将"ready to execute"折叠为"readytoexecute"）。

### 你现在可以做什么

- **在 Conductor 中使用计划模式审查技能。** 打开 Conductor 工作区，针对计划运行 `/plan-ceo-review`，范围模式问题实际上会出现供你回答。`/plan-eng-review`、`/plan-design-review`、`/plan-devex-review`、`/autoplan` 的前提门和 `/office-hours` 也是如此。
- **在 `--disallowedTools` 下保持控制，无需编写模板覆盖。** 工具解析部分位于每个层级 ≥2 技能的前言位置 1；通过相同模式禁用原生 AUQ 的新主机只要注册 MCP 变体就可以透明地获得修复。
- **选择加入 AUTO_DECIDE 而不会失去回归守卫。** 为特定问题设置 `never-ask` 的 `/plan-tune` 用户在 Conductor 标志下保持自动选择；周期性层 `auto-decide-preserved` 评估保护此路径。

### 重要的数字

来源：`ps -p <conductor-claude-pid> -o args=` 用于回归机制（验证主要来源）。6 个新的门层回归案例 + 1 个周期性层 AUTO_DECIDE 评估；覆盖在 `test/skill-e2e-plan-{ceo,eng,design,devex}-plan-mode.test.ts`（参数化内联）+ `test/skill-e2e-{autoplan,office-hours}-auto-mode.test.ts`（独立）+ `test/skill-e2e-auto-decide-preserved.test.ts`（周期性）。

| 表面 | 形状 |
|---|---|
| 在 Conductor 中重新获得交互性的技能 | 6（`/plan-ceo-review`、`/plan-eng-review`、`/plan-design-review`、`/plan-devex-review`、`/autoplan`、`/office-hours`） |
| 新的门层回归测试案例 | 6（每个技能一个；`--disallowedTools AskUserQuestion` 参数化） |
| 新的周期性层评估 | 1（`auto-decide-preserved`，保护 `/plan-tune` 选择加入路径） |
| 新的 `ClassifyResult` 结果 | `auto_decided` — TTY 显示"自动决策...（你的偏好）" |
| 新的 `runPlanSkillObservation` 参数 | `extraArgs?: string[]` — 将原始标志传递给生成的 `claude` |
| 触及的前言解析器 | 2（`generate-ask-user-format.ts`、`generate-completion-status.ts`） |
| 重新生成的 SKILL.md 文件 | 41 |
| `classifyVisible` 分支顺序 | `silent_write` → `auto_decided` → `plan_ready` → `asked`（每个比下一个更具体） |
| 容忍空格的检测器 | `isPlanReadyVisible`、`isAutoDecidedVisible`（击败 stripAnsi 光标定位折叠） |
| 验证者 | `ps -p <conductor-claude-pid> -o args=` 显示 `--disallowedTools AskUserQuestion --permission-mode default` |

### 这对构建者意味着什么

如果你在此版本之前在 Conductor 中运行了 `/plan-ceo-review` 或任何计划模式审查技能，该技能会静默生成一个你没有塑造的计划 — 范围模式问题、扩展提案和每节 STOP 从未到达你。升级后，技能会在模板定义的每个门处停止。修复在前言中，因此你不需要自己更新技能模板 — 只需升级 gstack，你运行的下一个计划审查就会尊重你的输入。

如果你通过 `/plan-tune` 选择加入自动决策特定问题，周期性评估会保护该路径。修复是"在注册时优先选择 MCP 变体"，而不是"强制每个问题浮现"— 你的 `never-ask` 偏好仍然自动选择，AUTO_DECIDE 注释仍然呈现，对于选择加入用户没有任何变化。

gstack 端回归测试表面现在反映了真实用户遇到的情况。每个计划模式测试文件都获得了第二个 `test()` 块，设置 `extraArgs: ['--disallowedTools', 'AskUserQuestion']` 并断言 AskUserQuestion 仍然浮现。建立在 v1.21.1.0 的 `classifyVisible()` 提取之上 — 新的自动决策分支在 silent_write 和 plan_ready 之间干净地插入。

### 逐项更改

#### 添加 — 工具解析前言

- `scripts/resolvers/preamble/generate-ask-user-format.ts` 在 AskUserQuestion 格式块的顶部获得一个新的 `### Tool resolution (read first)` 部分。告诉模型：AskUserQuestion 可以在运行时解析为两个工具（主机 MCP 变体或原生）；优先选择工具列表中的任何 `mcp__*__AskUserQuestion` 变体而不是原生；主机可能通过 `--disallowedTools AskUserQuestion` 禁用原生（Conductor 默认这样做）；相同的问题/选项形状和决策简报格式适用于 MCP 变体。包括当两个变体都不可调用时的回退路径：将决策作为 `## Decisions to confirm` + ExitPlanMode 写入计划文件。
- `scripts/resolvers/preamble/generate-completion-status.ts`（前言位置 1 的计划模式信息块）更新以指向工具解析部分：AskUserQuestion 满足计划模式的"任何变体"回合结束要求，对于无变体情况有计划文件回退。

#### 添加 — 回归测试

- 4 个内联 `test()` 块添加到 `test/skill-e2e-plan-{ceo,eng,design,devex}-plan-mode.test.ts`。每个都使用 `extraArgs: ['--disallowedTools', 'AskUserQuestion']` 生成 claude 并断言技能仍然浮现问题 — 通过信封 `['asked', 'plan_ready']`（后者涵盖计划文件回退流程），失败信号是 `'auto_decided'`（明确捕获）加上标准的 silent_write/exited/timeout。
- `test/skill-e2e-autoplan-auto-mode.test.ts`（新）。断言 autoplan 的第一个非自动决策门（第 1 阶段前提确认）仍然浮现。Autoplan 按设计自动决策中间问题，因此测试范围限定在用户必须看到的门。
- `test/skill-e2e-office-hours-auto-mode.test.ts`（新）。断言 office-hours 的启动与构建者模式 AskUserQuestion 仍然浮现。
- `test/skill-e2e-auto-decide-preserved.test.ts`（新，周期性层）。设置一个隔离的 `GSTACK_HOME` tmpdir，写入 `question_tuning=true` + `plan-ceo-review-mode` 的 `never-ask` 偏好（来源 `'plan-tune'`），在 `--disallowedTools AskUserQuestion` 下运行 `/plan-ceo-review`，断言结果不是 `'asked'`（模型尊重选择加入）。

#### 更改 — PTY 线束

- `test/helpers/claude-pty-runner.ts`：`runPlanSkillObservation` 接受新的可选 `extraArgs?: string[]`（直接传递给 `launchClaudePty`，它已经支持该字段）。`ClassifyResult` 获得 `'auto_decided'` 结果加上 `isAutoDecidedVisible(visible)` 检测器，匹配 AUTO_DECIDE 前言模板（`Auto-decided ... (your preference)`）。`classifyVisible` 分支顺序扩展为 `silent_write → auto_decided → plan_ready → asked`，因此上游自动决策不会被下游计划模式确认掩盖。
- 容忍空格的检测：`isPlanReadyVisible` 和 `isAutoDecidedVisible` 现在测试其目标短语的间隔和空格折叠形式。`stripAnsi` 删除光标定位转义（`\x1b[40C`）而不用空格替换它们，因此"ready to execute"可以作为"readytoexecute"出现 — 间隔正则表达式会错过它。

#### 更改 — touchfiles

- `test/helpers/touchfiles.ts`：现有的 `plan-X-review-plan-mode` 条目获得 `scripts/resolvers/question-tuning.ts` 和 `scripts/resolvers/preamble/generate-ask-user-format.ts` 作为 touchfile 依赖项，因此带有 AUTO_DECIDE 的解析器更改会正确使回归案例无效。
- 新条目：`autoplan-auto-mode`（门）、`office-hours-auto-mode`（门）、`auto-decide-preserved`（周期性）。
- `test/touchfiles.test.ts`：`plan-ceo-review/SKILL.md` 选择的测试计数从 19 更新到 21，以涵盖依赖于 `plan-ceo-review/**` 的新条目。

#### 对于贡献者

- PTY 线束的 `auto_decided` 结果是深度防御信号：它在 AUTO_DECIDE 前言模板措辞上触发，这是非确定性的。将其视为回归的证据，而不是硬性合同。
- 工具解析部分是任何未来以类似方式禁用原生 AUQ 的主机的手术修复站点。模式：注册一个 `mcp__<host>__AskUserQuestion` MCP 工具；gstack 前言已经告诉模型优先选择它。每个主机不需要技能模板更改。
- `auto-decide-preserved` 在隔离的 `GSTACK_HOME` tmpdir 中运行，以避免改变开发人员的真实 `~/.gstack` 状态。调试时，手动将 `GSTACK_HOME` 设置为临时目录，并运行测试执行的相同设置（`gstack-config set question_tuning true`，然后 `gstack-question-preference --write`）。

## [1.24.0.0] - 2026-04-30

## **跨平台加固。Mac + Linux 完整，添加了精选的 Windows 通道。**

v1.24.0.0 将 McGluut 分支的可移植性工作移植到上游，并添加了一个实际运行绿色的精选 Windows 测试作业。`bin/gstack-paths` 通过一个从技能 bash 块通过 `eval "$(...)"` 源化的助手整合状态根解析；八个技能（`careful`、`freeze`、`guard`、`unfreeze`、`investigate`、`context-save`、`context-restore`、`learn`、`office-hours`、`plan-tune`、`codex`）从内联 `${CLAUDE_PLUGIN_DATA:-...}` 链中移出。`Bun.which()` 在新的 `browse/src/claude-bin.ts` 包装器中替换了 75 行分支端 PATH 解析代码，通过五个硬编码的 `claude` 生成站点连接。一个新的 `windows-free-tests` GitHub Actions 作业在 `windows-latest` 上运行精选的 103 个测试子集加上目标解析器测试；`evals.yml` 保持 Linux 容器，正如它应该的那样。`AGENTS.md` 和 `docs/skills.md` 同步到实时技能清单（40+ 技能，之前是 21）；`/debug` → `/investigate`，添加了缺失的技能，删除了过时的 `<5s` `bun test` 声明。加固方向归功于 McGluut