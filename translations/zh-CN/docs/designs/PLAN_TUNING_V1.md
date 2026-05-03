# 计划调优 v1 — 设计文档

**状态:** 已批准实施 (2026-04-18)
**分支:** garrytan/plan-tune-skill
**作者:** Garry Tan (用户), 由 Claude Opus 4.7 + OpenAI Codex gpt-5.4 协助审查
**取代范围:** 在 [PLAN_TUNING_V0.md](./PLAN_TUNING_V0.md) (观察基础设施) 之上添加写作风格 + LOC 收据层。V0 保持不变。
**相关:** [PACING_UPDATES_V0.md](./PACING_UPDATES_V0.md) — 提取的节奏改进,V1.1 计划。

## 本文档的内容

这是 /plan-tune v1 的规范记录,包括它是什么、不是什么、我们考虑了什么,以及为什么做出每个决定。提交到代码库,以便未来的贡献者(和未来的 Garry)可以追溯推理过程而无需考古。取代任何用户本地计划工件。

## 致谢

这个计划的存在要归功于 **[Louise de Sadeleer](https://x.com/LouiseDSadeleer/status/2045139351227478199)**,她作为非技术用户完整体验了一次 gstack 运行,并告诉我们真实的感受。她的具体反馈:

1. "过了一会儿我有点累了,感觉有点僵硬。" — *节奏/疲劳*
2. "我只会说是是是"(在架构审查期间)。 — *脱离参与*
3. "我觉得有趣的是他强调自己产出了多少行代码。当然是 AI 为他产出的。" — *LOC 框架*
4. "作为非工程师,这有点难以理解。" — *术语密度 + 结果框架*

V1 直接解决 #3 和 #4:术语解释 + 结果导向的写作,读起来像真人为读者写的,加上可辩护的 LOC 重新框架。Louise 的 #1 和 #2(节奏/疲劳)需要单独的设计轮次 — 提取到 [PACING_UPDATES_V0.md](./PACING_UPDATES_V0.md) 作为 V1.1 计划。

## 一段话概括功能

gstack skill 输出就是产品。如果文案对非技术创始人来说读起来不好,他们会退出审查并点击"是是是"。V1 为每个 ≥ 2 级的 skill 添加了写作风格标准:首次使用时解释术语(来自精选的约 50 个术语列表),用结果术语("如果...对你的用户有什么影响")而非实现术语来框架问题,短句,具体名词。想要更紧凑的 V0 文案的高级用户可以设置 `gstack-config set explain_level terse`。二进制开关,没有部分模式。另外:README 中的"600,000+ 行生产代码"框架 — 被 Louise 正确指出为 LOC 虚荣 — 被替换为基于 `scc` 支持的脚本计算的真实 2013 对比 2026 按比例倍数,并诚实说明公共与私有仓库可见性的注意事项。

## 为什么我们构建更小的版本

V1 经历了四次重大范围修订,经过多轮审查。最终范围比任何中间版本都小,因为每轮审查都发现了真实问题。

**修订 1 — 四级体验轴(拒绝)。** 原始提案:在首次运行时询问用户是否是有经验的开发者、无独立经验的工程师、在团队中交付过的非技术人员,或完全非技术人员。Skills 根据级别调整。在 CEO 审查的前提挑战步骤中被拒绝,因为 (a) 入门询问在 V1 试图减少摩擦的时刻增加了摩擦,(b) "我是什么级别?"对最需要帮助的用户来说本身就是一个令人困惑的问题,(c) 技术专长不是一维的(设计师在 CSS 上是 A 级,在部署上是 D 级),(d) 工程师也能从非技术用户需要的相同写作标准中受益。

**修订 2 — 默认 ELI10,简洁选择退出(接受)。** 每个 skill 的输出默认使用写作标准。想要 V0 文案的高级用户设置 `explain_level: terse`。Codex Pass 1 发现了关键缺口(静态 markdown 门控、主机感知路径、README 更新机制)— 全部三个都已集成。

**修订 3 — ELI10 + 审查节奏改进(提议,缩减范围)。** 添加了节奏工作流:对发现进行排名,自动接受双向门,每个阶段最多 3 个 AskUserQuestion 提示,带翻转命令的 Silent Decisions 块。旨在直接解决 Louise 的 #1 和 #2。工程审查 Pass 2 发现了评分公式和路径一致性错误。工程审查 Pass 3 + Codex Pass 2 发现了节奏工作流中 10+ 个无法通过计划文本编辑修复的结构性缺口。

**修订 4 — 仅 ELI10 + LOC(最终)。** 用户选择缩减范围:V1 交付写作风格 + LOC 收据,将节奏推迟到 V1.1,通过 [PACING_UPDATES_V0.md](./PACING_UPDATES_V0.md)。这是批准的 V1 范围。

主线:每轮审查都正确缩小了野心,直到剩余范围没有结构性缺口。符合 CEO 审查 skill 的 SCOPE REDUCTION 模式,通过工程审查后期到达,而非通过战略选择早期到达。

## v1 范围(我们现在正在构建的)

1. **前言中的写作风格部分** (`scripts/resolvers/preamble.ts`)。六条规则:每次 skill 调用时首次使用术语时解释,结果框架,短句/具体名词/主动语态,决策以用户影响结束,无条件首次使用解释(即使用户粘贴了该术语),用户回合覆盖(用户说"简洁点"→ 跳过该响应)。
2. **通过仓库拥有的列表划定术语边界** (`scripts/jargon-list.json`)。约 50 个精选的高频技术术语。列表中没有的术语被假定为足够通俗易懂。术语在 `gen-skill-docs` 时内联到生成的 SKILL.md 文案中(零运行时成本)。
3. **简洁选择退出** (`gstack-config set explain_level terse`)。二进制:`default` 对比 `terse`。Terse 完全跳过写作风格块并使用 V0 文案风格。
4. **主机感知前言回显。** `_EXPLAIN_LEVEL=$(${binDir}/gstack-config get explain_level 2>/dev/null || echo "default")`。通过现有 V0 `ctx.paths.binDir` 模式实现主机可移植性。
5. **gstack-config 验证。** 在头部记录 `explain_level: default|terse`。白名单值。对未知值发出警告,带有特定消息 + 默认为 `default`。
6. **README 中的 LOC 重新框架。** 删除"600,000+ 行生产代码"英雄框架。插入 `<!-- GSTACK-THROUGHPUT-PLACEHOLDER -->` 锚点。构建时脚本用计算的倍数 + 注意事项替换锚点。
7. **基于 `scc` 的吞吐量脚本** (`scripts/garry-output-comparison.ts`)。对于 2013 + 2026 的每一年,枚举 Garry 编写的公共提交,从 `git diff` 提取添加的行,通过 `scc --stdin`(或正则回退)分类。输出 `docs/throughput-2013-vs-2026.json`,包含每种语言的细分 + 注意事项。
8. **`scc` 作为独立安装脚本** (`scripts/setup-scc.sh`)。不是 `package.json` 依赖项(真正可选 — 95% 的用户从不运行吞吐量)。检测操作系统并运行 `brew install scc` / `apt install scc` / 打印 GitHub 发布链接。
9. **README 更新管道** (`scripts/update-readme-throughput.ts`)。如果存在则读取 `docs/throughput-2013-vs-2026.json`,用计算的数字替换锚点。如果缺失,写入 `GSTACK-THROUGHPUT-PENDING` 标记,CI 拒绝 — 强制贡献者在提交前运行脚本。
10. **/retro 在原始 LOC 之上添加逻辑 SLOC + 加权提交。** 原始 LOC 保留用于上下文,但在视觉上降级。
11. **升级迁移** (`gstack-upgrade/migrations/v<VERSION>.sh`)。升级后一次性交互式提示,为喜欢 V0 文案的用户提供通过 `explain_level: terse` 恢复的选项。标志文件门控。
12. **文档。** CLAUDE.md 获得写作风格部分(项目约定)。CHANGELOG.md 获得 V1 条目(面向用户的叙述,提及范围缩减 + V1.1 节奏)。README.md 获得写作风格解释部分(约 80 字)。CONTRIBUTING.md 获得关于术语列表维护的说明(添加/删除术语的 PR)。
13. **测试。** 6 个新测试文件 + 扩展现有的 `gen-skill-docs.test.ts`。除 LLM-judge E2E(定期)外,所有测试都门控层级。
14. **V0 休眠负面测试。** 断言 5D 维度名称和 8 个原型名称不出现在默认模式 skill 输出中。防止 V0 心理特征机制泄漏到 V1。
15. **V1 和 V1.1 设计文档。** PLAN_TUNING_V1.md(本文件)。PACING_UPDATES_V0.md(V1.1 计划,在 V1 实施期间从提取的附录创建)。TODOS.md P0 条目。

## 推迟

**到 V1.1(明确,有专门的设计文档):**
- 审查节奏改进(排名、自动接受、每阶段最多 3 个、Silent Decisions 块、翻转机制)。理由:见 [PACING_UPDATES_V0.md](./PACING_UPDATES_V0.md) §"为什么提取"。有 10+ 个无法通过仅文案更改修复的结构性缺口。
- 前言首次运行元提示审计(lake 介绍、遥测、主动、路由)。Louise 在首次运行时看到了所有这些;它们计入疲劳。V1.1 考虑抑制直到会话 N。

**到 V2(或更晚):**
- 从问题日志驱动的即时翻译提供的混淆信号检测。
- 5D 心理特征驱动的 skill 适应(V0 E1 项)。
- /plan-tune 叙述 + /plan-tune 氛围(V0 E3 项)。
- 每个 skill 或每个主题的解释级别。
- 团队配置文件。
- 基于 AST 的"交付功能"指标。

## 完全拒绝(考虑过,不做)

- **四级声明的体验轴(A/B/C/D)。** 在 CEO 审查前提挑战期间被拒绝。见上文"为什么我们构建更小的版本"。
- **ELI10 作为新的解析器文件(`scripts/resolvers/eli10-writing.ts`)。** Codex Pass 1 发现与前言的 AskUserQuestion Format 部分中现有的"聪明的 16 岁"框架冲突。改为折叠到现有前言中。
- **写作风格块的运行时抑制。** Codex Pass 1 发现 `gen-skill-docs` 生成静态 Markdown — 运行时 `EXPLAIN_LEVEL=terse` 无法隐藏已经烘焙的内容。解决方案:条件文案门控(文案约定,与 V0 的 `QUESTION_TUNING` 门控同类)。
- **默认和简洁之间的中间写作模式。** 修订 3 提议"terse = 无解释但保留结果框架"。Codex Pass 2 发现与迁移消息的矛盾。二进制获胜:terse = V0 文案,完全停止。
- **运行时用户可编辑的术语列表。** 修订 3 提议 `~/.gstack/jargon-list.json` 作为用户覆盖。Codex Pass 2 发现与生成时内联的矛盾。解决:仅仓库拥有,PR 添加/删除,重新生成以生效。
- **package.json 中的 `devDependencies.optional` 字段。** 不是真正的 npm/bun 字段。工程审查 Pass 2 发现。改为独立安装脚本。
- **在 README 中使用相同字符串作为替换锚点和 CI 拒绝标记。** 工程审查 Pass 2 / Codex Pass 2 发现这会使管道破坏自己的更新路径。双字符串解决方案:`GSTACK-THROUGHPUT-PLACEHOLDER`(锚点,跨运行保持)对比 `GSTACK-THROUGHPUT-PENDING`(明确的"构建未运行"标记,CI 拒绝)。
- **"每个技术术语都得到解释"作为验收标准。** Codex Pass 2 发现与精选列表规则的矛盾。验收重写以匹配规则:"出现在 `scripts/jargon-list.json` 上的每个术语都得到解释"。
- **验收标准"每个 /autoplan ≤ 12 个 AskUserQuestion 提示"。** 从 V1 中删除 — 该目标需要现在在 V1.1 中的节奏改进。

## 架构

```
~/.gstack/
  developer-profile.json           # 与 V0 相比未更改
  config.yaml                       # + explain_level 键(default | terse)

scripts/
  jargon-list.json                  # 新:约 50 个仓库拥有的术语(生成时内联)
  garry-output-comparison.ts        # 新:scc + git 每年,作者范围
  update-readme-throughput.ts       # 新:README 锚点替换
  setup-scc.sh                      # 新:操作系统检测 scc 安装程序
  resolvers/preamble.ts             # 修改:写作风格部分 + EXPLAIN_LEVEL 回显

docs/
  designs/PLAN_TUNING_V1.md         # 新:本文件
  designs/PACING_UPDATES_V0.md      # 新:V1.1 计划(提取)
  throughput-2013-vs-2026.json      # 新:计算,已提交

~/.claude/skills/gstack/bin/
  gstack-config                     # 修改:explain_level 头部 + 验证

gstack-upgrade/migrations/
  v<VERSION>.sh                     # 新:V0 → V1 交互式提示
```

### 数据流

```
用户运行 ≥2 级 skill
       │
       ▼
前言 bash(每次调用):
  _EXPLAIN_LEVEL=$(${binDir}/gstack-config get explain_level 2>/dev/null || "default")
  echo "EXPLAIN_LEVEL: $_EXPLAIN_LEVEL"
       │
       ▼
生成的 SKILL.md 主体(静态 Markdown,在 gen-skill-docs 时烘焙):
  - AskUserQuestion Format 部分(现有 V0)
  - 写作风格部分(新,条件文案门控)
       │
       ├── "如果 EXPLAIN_LEVEL: terse 或用户本回合说'简洁点'则跳过"
       ├── 6 条写作规则(术语、结果、简短、影响、首次使用、覆盖)
       └── 从 scripts/jargon-list.json 内联的术语列表
       │
       ▼
Agent 根据运行时 EXPLAIN_LEVEL + 用户回合信号应用或跳过
       │
       ▼
V0 QUESTION_TUNING + question-log + preferences 未更改
       │
       ▼
输出给用户(首次使用解释、结果框架、短句;或如果 terse 则为 V0 文案)
```

### 数据流:吞吐量脚本(构建时)

```
bun run build
   │
   ├── gen:skill-docs(重新生成内联术语列表的 SKILL.md 文件)
   ├── update-readme-throughput(如果存在则读取 JSON;替换锚点或写入 PENDING 标记)
   └── 其他步骤(二进制编译等)

单独,按需:
bun run scripts/garry-output-comparison.ts
   │
   ├── scc 预检(如果缺失 → 退出并提示 setup-scc.sh)
   ├── 对于 2013 + 2026:枚举公共 garrytan/* 仓库中 Garry 编写的提交
   ├── 对于每个提交:git diff,提取添加的行,通过 scc --stdin 分类
   └── 写入 docs/throughput-2013-vs-2026.json(每种语言 + 注意事项)
```

## 安全 + 隐私

- **无新用户数据。** V1 扩展前言文案 + 配置键。不收集新的个人数据。
- **无敏感数据的运行时文件读取。** 术语列表是仓库提交的精选列表。
- **迁移脚本是一次性的。** 标志文件防止重新触发。
- **scc 仅在公共仓库上运行。** 无法访问私有工作。

## 决策日志(带利弊)

### 决策 A:四级体验轴 vs. 默认 ELI10 — 答案:默认 ELI10

**四级轴(拒绝):** 要求用户在首次运行时自我识别为 A/B/C/D。Skills 根据级别调整。
- 优点:明确的用户主权。高级用户获得 V0 行为。
- 缺点:增加入门摩擦。强迫用户给自己贴标签。技术专长不是一维的。工程师也能从非技术用户需要的相同写作标准中受益。

**默认 ELI10,简洁选择退出(选择):** 每个 skill 的输出默认使用写作标准。高级用户设置 `explain_level: terse`。
- 优点:无入门问题。良好的写作使每个人受益。高级用户仍有逃生舱口。
- 缺点:在升级时静默更改 V0 行为 → 需要迁移提示。

### 决策 B:新解析器文件 vs. 扩展现有前言 — 答案:扩展现有

**新解析器(拒绝):** `scripts/resolvers/eli10-writing.ts` 作为单独的生成器。
- 优点:模块化。
- 缺点(Codex #7):与前言的 AskUserQuestion Format 部分中现有的"聪明的 16 岁"框架冲突。两个真相来源。

**扩展前言(选择):** 写作风格部分直接添加到 `scripts/resolvers/preamble.ts`,位于 AskUserQuestion Format 下方。
- 优点:一个真相来源。与现有规则组合。
- 缺点:`preamble.ts` 增长。

### 决策 C:运行时抑制 vs. 条件文案门控 — 答案:条件文案门控

**运行时抑制(拒绝):** 前言读取 `explain_level` 触发抑制逻辑。
- 优点:更简单的心智模型。
- 缺点(Codex #1):`gen-skill-docs` 生成静态 Markdown。一旦烘焙,内容无法追溯隐藏。运行时抑制是虚构的。

**条件文案门控(选择):** "如果 EXPLAIN_LEVEL: terse 或用户本回合说'简洁点'则跳过此块"。文案约定;agent 在运行时遵守或不遵守。
- 优点:可测试。匹配 V0 的 `QUESTION_TUNING` 模式。对机制诚实。
- 缺点:依赖 agent 文案合规性(无硬运行时门控)。

### 决策 D:术语列表位置 — 运行时用户可编辑 vs. 仓库拥有生成时 — 答案:仓库拥有生成时

**运行时用户可编辑(拒绝):** `~/.gstack/jargon-list.json` 覆盖 `scripts/jargon-list.json`。
- 优点:用户可以添加特定于其领域的术语。
- 缺点(Codex #4, Pass 2):生成时内联意味着用户编辑需要重新生成。矛盾。

**仓库拥有,生成时内联(选择):** 仅 `scripts/jargon-list.json`。PR 添加/删除。`bun run gen:skill-docs` 将术语内联到前言文案中。
- 优点:一个真相来源。零运行时成本。与现有构建可组合。
- 缺点:用户无法在本地添加术语。缓解:在 CONTRIBUTING.md 中记录;接受 PR。

### 决策 E:V1 中的节奏改进 vs. V1.1 — 答案:V1.1(提取)

**V1 中的节奏(拒绝):** 捆绑排名 + 自动接受 + Silent Decisions + 每阶段最多 3 个上限 + 翻转机制。
- 优点:直接解决 Louise 的疲劳。
- 缺点(工程审查 Pass 3 + Codex Pass 2):10+ 个无法通过计划文本编辑修复的结构性缺口。会话状态模型未定义。question-log 中缺少 `phase` 字段。注册表不涵盖动态审查发现。翻转机制没有实现。迁移提示本身就是中断。首次运行前言提示也计数。作为文案的节奏无法反转现有的每节询问执行顺序。

**提取到 V1.1(选择):** V1 交付 ELI10 + LOC。节奏获得自己的设计轮次,完整审查周期。
- 优点:诚实地交付 V1。为 V1.1 提供来自 V1 使用的真实基线数据(Louise 的 V1 记录)。匹配 CEO 审查的 SCOPE REDUCTION 模式。
- 缺点:Louise 的疲劳投诉直到 V1.1 才完全解决。缓解:V1 仍通过写作质量改善她的体验;V1.1 跟进节奏。

### 决策 F:README 更新机制 — 单字符串 vs. 双字符串 — 答案:双字符串

**单字符串(拒绝):** `<!-- GSTACK-THROUGHPUT-MULTIPLE: N× -->` 既作为替换锚点又作为 CI 拒绝标记。
- 优点:简单。
- 缺点(Codex Pass 2):管道在自身上中断 — CI 拒绝包含标记的提交,但标记就是锚点。

**双字符串(选择):** `GSTACK-THROUGHPUT-PLACEHOLDER`(锚点,稳定)+ `GSTACK-THROUGHPUT-PENDING`(明确的缺失构建标记,CI 拒绝)。
- 优点:锚点持久;CI 捕获实际失败状态。
- 缺点:要记住两个符号。

## 审查记录

| 审查 | 运行 | 状态 | 集成的关键发现 |
|---|---|---|---|
| CEO 审查 | 1 | CLEAR (HOLD SCOPE) | 前提转向:四级轴 → 默认 ELI10。通过明确的用户选择解决跨模型紧张关系。 |
| Codex 审查 | 2 | ISSUES_FOUND + 驱动范围缩减 | Pass 1:25 个发现,3 个关键阻塞(静态 markdown、主机路径、README 机制)。Pass 2:修订计划上的 20 个发现,驱动 V1.1 提取。 |
| 工程审查 | 3 | CLEAR (SCOPE_REDUCED) | Pass 1:关键缺口 + 3 个决策(全部 A)。Pass 2:评分公式错误、路径矛盾、假 `devDependencies.optional` 字段。Pass 3:识别节奏结构性缺口,驱动提取。 |
| DX 审查 | 1 | CLEAR (TRIAGE) | 3 个关键(文档计划、升级迁移、英雄时刻)。9 个自动接受为 Silent DX Decisions。 |

审查报告通过 `gstack-review-log` 持久化在 `~/.gstack/` 中。计划文件保留在 `~/.claude/plans/system-instruction-you-are-working-transient-sunbeam.md`,包含完整历史。