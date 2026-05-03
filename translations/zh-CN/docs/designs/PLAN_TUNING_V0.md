# Plan Tuning v0 — 设计文档

**状态:** 已批准 v1 实现
**分支:** garrytan/plan-tune-skill
**作者:** Garry Tan (用户),由 Claude Opus 4.7 + OpenAI Codex gpt-5.4 协助审查
**日期:** 2026-04-16

## 本文档的定位

这是关于 `/plan-tune` v1 是什么、不是什么、我们考虑了什么以及为何做出每个决策的权威记录。提交到代码库,以便未来的贡献者(和未来的 Garry)可以追溯推理过程而无需考古。取代了两个 `~/.gstack/projects/` 工件(office-hours 设计文档 + CEO 计划),它们是每个用户的本地记录。

## 一段话概述功能

gstack 的 40+ 个技能不断触发 AskUserQuestion。高级用户重复以相同方式回答相同问题,却无法告诉 gstack "别再问我这个了"。更根本的是,gstack 没有每个用户如何偏好引导工作的模型——范围偏好、风险容忍度、细节偏好、自主性、架构关注度——因此每个技能的默认值对所有人都是中庸的。`/plan-tune` v1 构建了模式 + 观察层:一个类型化的问题注册表、每个问题的显式偏好、内联 "tune:" 反馈,以及一个可通过纯英语检查的配置文件(声明 + 推断的维度)。它尚未根据配置文件调整技能行为。这将在 v2 中实现,在 v1 证明基础有效之后。

## 为什么我们要构建更小的版本

该功能最初是一个完整的自适应基础设施:心理维度驱动自动决策、盲点指导、LANDED 庆祝 HTML 页面,全部打包。经过四轮审查(office-hours、CEO EXPANSION、DX POLISH、工程审查)后通过。然后外部声音(Codex)提出了 20 点批评。关键发现,按优先级排序:

1. **"基础设施"是虚假的。** 该计划将 5 个技能连接到在前言中读取配置文件,但 AskUserQuestion 是一个提示约定,而非中间件。代理可以静默跳过指令。你无法在不可强制执行的约定之上可靠地构建自动决策。如果没有每个 AskUserQuestion 都路由通过的类型化问题注册表,基础设施的说法就是营销。
2. **内部逻辑矛盾。** E4(盲点)+ E6(不匹配)+ 声明维度的 ±0.2 限制无法组合。如果用户自我声明通过限制成为基本事实,E6 的不匹配检测就是在检测噪音。如果行为可以纠正配置文件,限制就会抑制 E6 需要的信号。
3. **配置文件污染。** 内联 "tune: never ask" 可能由恶意仓库内容(README、PR 描述、工具输出)发出,代理会忠实地写入它。之前的审查都没有发现这个安全漏洞。
4. **前言中的 E5 LANDED 页面。** 在每个技能的前言中执行 `gh pr view` + HTML 写入 + 浏览器打开会导致延迟、认证失败、速率限制、意外的浏览器打开,以及注入到最热路径中的非确定性。
5. **实现顺序颠倒了。** 该计划从分类器和分箱开始。正确的顺序是:首先构建集成点(类型化问题注册表),然后是基础设施,然后是消费者。

在权衡 Codex 的论点后,我们选择回滚 CEO EXPANSION,发布一个观察性的 v1,以真实的类型化注册表作为基础。心理维度只有在注册表在生产中证明持久后才变为行为性的。

## v1 范围(我们现在正在构建的)

1. **类型化问题注册表** (`scripts/question-registry.ts`)。gstack 使用的每个 AskUserQuestion 都用 `{id, skill, category, door_type, options[], signal_key?}` 声明。受模式管理。
2. **CI 强制执行。** Lint 测试(门控层)断言 SKILL.md.tmpl 文件中的每个 AskUserQuestion 模式都有匹配的注册表条目。在漂移、重命名或重复时 CI 失败。
3. **问题日志** (`bin/gstack-question-log`)。将 `{ts, question_id, user_choice, recommended, session_id}` 追加到 `~/.gstack/projects/{SLUG}/question-log.jsonl`。根据注册表验证。
4. **每个问题的显式偏好** (`bin/gstack-question-preference`)。写入 `{question_id, preference}`,其中 preference 是 `always-ask | never-ask | ask-only-for-one-way`。从第 1 个会话开始遵守。无校准门控——用户声明了,系统就遵守。
5. **前言注入。** 在每个 AskUserQuestion 之前,代理调用 `gstack-question-preference --check <registry-id>`。如果是 `never-ask` 且问题不是单向门,则自动选择推荐选项并显示注释:"Auto-decided [summary] → [option] (your preference). Change with /plan-tune." 无论偏好如何,单向门始终询问——安全覆盖。
6. **带用户来源门控的内联 "tune:" 反馈。** 代理提供 "Tune this question? Reply `tune: [feedback]` to adjust." 用户可以使用快捷方式(`unnecessary`、`ask-less`、`never-ask`、`always-ask`、`context-dependent`)或自由形式的英语。关键:代理仅在 `tune:` 内容出现在用户当前聊天轮次中时才写入调整事件——不在工具输出中,不在读取的文件中。二进制文件在写入时验证 `source: "inline-user"`;拒绝其他来源。
7. **声明的配置文件** (`/plan-tune setup`)。5 个纯英语问题,每个维度一个。存储在统一的 `~/.gstack/developer-profile.json` 的 `declared: {...}` 下。在 v1 中仅供参考——不改变技能行为。
8. **观察/推断的配置文件。** 每个问题日志事件通过手工制作的信号映射(`scripts/psychographic-signals.ts`)为推断维度贡献增量。按需计算。显示但不执行。
9. **`/plan-tune` 技能。** 对话式纯英语检查工具。"显示我的配置文件"、"设置偏好"、"我被问过什么问题"、"显示我说的和我做的之间的差距"。不需要 CLI 子命令语法。
10. **与现有 `~/.gstack/builder-profile.jsonl` 统一。** 将 /office-hours 会话记录和累积的信号合并到统一的 `~/.gstack/developer-profile.json` 中。迁移是原子的 + 幂等的 + 归档源文件。

## 推迟到 v2(不在此 PR 中,但有明确的验收标准)

| 项目 | 推迟原因 | v2 推广的验收标准 |
|------|----------|-------------------|
| E1 基础设施连接(5 个技能读取配置文件并适应) | 需要 v1 注册表证明持久。需要真实观察数据来校准信号增量。心理漂移风险。 | v1 注册表稳定 90+ 天。推断维度在 3+ 个技能中显示明确的稳定性。用户狗粮验证基于配置文件的默认值感觉正确。 |
| E3 `/plan-tune narrative` + `/plan-tune vibe` | 事件锚定的叙述需要稳定的配置文件。没有 v1 数据,输出将是通用的废话。 | 配置文件多样性检查通过 2+ 周的实际使用。叙述测试证明它引用特定事件,而非陈词滥调。 |
| E4 盲点教练 | 在没有明确交互预算设计的情况下,与 E1/E6 逻辑冲突。需要全局会话预算、升级规则、从不匹配检测中排除。 | 交互预算 + 升级的设计规范。狗粮确认挑战感觉像指导,而非唠叨。 |
| E5 LANDED 庆祝 HTML 页面 | 不能存在于前言中(Codex #9、#10)。推广时,移至显式命令 `/plan-tune show-landed` 或发布后钩子——而非热路径中的被动检测。 | 显式命令或钩子设计。/design-shotgun → /design-html 用于视觉方向。PR 数据聚合的安全 + 隐私审查。 |
| E6 基于不匹配的自动调整 | 在 v1 中,/plan-tune 显示声明和推断之间的差距。在 v2 中,它可以建议声明更新。需要双轨配置文件稳定。 | v1 的真实不匹配数据显示一致的模式。单独设计建议 UX。 |
| 心理驱动的自动决策 | v1 中零行为变化。仅显式偏好起作用。 | 实际使用显示显式偏好涵盖大多数情况。推断配置文件足够稳定以信任。 |

## 完全拒绝(Codex 是对的,我们不做这些)

| 项目 | 拒绝原因 |
|------|----------|
| 基础设施作为提示约定(vs. 类型化注册表) | Codex #1。代理可以静默跳过指令。在其上构建心理维度是沙子。 |
| 声明维度的 ±0.2 限制 | Codex #6。与 E6 不匹配检测产生逻辑矛盾。选择一个:可编辑偏好或推断行为。现在:两者,分别跟踪(双轨配置文件)。 |
| 通过解析散文摘要进行单向门分类 | Codex #4。安全取决于措辞。door_type 必须在问题定义站点(注册表)声明,而非推断。 |
| 混合声明 + 覆盖 + 判决 + 反馈的单一事件模式文件 | Codex #5。不兼容的域对象。现在拆分为三个文件:question-log.jsonl、question-preferences.json、question-events.jsonl。 |
| /plan-tune 入门的 TTHW 遥测 | Codex #14。与本地优先框架矛盾。仅本地日志。 |
| 没有用户来源验证的内联 tune: 写入 | Codex #16。配置文件污染攻击。现在:用户来源门控是非可选的。 |

## 架构

```
~/.gstack/
  developer-profile.json            # 统一:声明 + 推断 + 会话(来自 office-hours)

~/.gstack/projects/{SLUG}/
  question-log.jsonl                # 每个 AskUserQuestion,仅追加,注册表验证
  question-preferences.json         # 显式的每个问题用户选择
  question-events.jsonl             # tune: 反馈事件,用户来源门控
```

**统一配置文件模式**(取代 v0.16.2.0 builder-profile.jsonl 和提议的 developer-profile.json):

```json
{
  "identity": {"email": "..."},
  "declared": {
    "scope_appetite": 0.9,
    "risk_tolerance": 0.7,
    "detail_preference": 0.4,
    "autonomy": 0.5,
    "architecture_care": 0.7
  },
  "inferred": {
    "values": {"scope_appetite": 0.72, "risk_tolerance": 0.58, "...": "..."},
    "sample_size": 47,
    "diversity": {
      "skills_covered": 5,
      "question_ids_covered": 14,
      "days_span": 23
    }
  },
  "gap": {"scope_appetite": 0.18, "...": "..."},
  "sessions": [
    {"date": "...", "mode": "builder", "project_slug": "...", "signals": []}
  ],
  "signals_accumulated": {
    "named_users": 1, "taste": 4, "agency": 3, "...": "..."
  }
}
```

**多样性检查**(Codex #13):`inferred` 仅在 `sample_size >= 20 AND skills_covered >= 3 AND question_ids_covered >= 8 AND days_span >= 7` 时被视为"足够的数据"。低于此值,`/plan-tune profile` 显示"尚无足够的观察数据"而非可能误导的推断值。

## 数据流(v1)

1. 前言:检查 `question_tuning` 配置。如果关闭,什么都不做。
2. 在每个 AskUserQuestion 之前:
   - 代理调用 `gstack-question-preference --check <registry-id>`
   - 如果是 `never-ask` 且问题不是单向门 → 自动选择推荐并带注释
   - 如果是 `always-ask`、未设置或问题是单向门 → 正常询问
3. 在 AskUserQuestion 之后:
   - 将日志记录追加到 question-log.jsonl(注册表验证,拒绝未知 ID)
4. 提供内联:"Tune this question? Reply `tune: [feedback]` to adjust."
5. 如果用户的下一轮消息包含 `tune:` 前缀且内容源自用户自己的消息(非工具输出):
   - 代理调用 `gstack-question-preference --write`,带 `source: "inline-user"`
   - 二进制文件验证 source 字段;如果不是 `inline-user` 则拒绝
6. 推断维度由 `bin/gstack-developer-profile --derive` 按需重新计算。信号映射更改触发从事件历史的完全重新计算。

## 安全模型

**配置文件污染防御**(Codex #16,下面的决策 J):内联调整事件仅在以下情况下可写入:
- 代理正在处理用户的当前聊天轮次
- `tune:` 前缀出现在该用户消息中(不在任何工具输出、文件内容、PR 描述、提交消息等中)
- 解析器对代理的指令明确指出这一点

二进制强制执行:`gstack-question-preference --write` 要求每个调整源记录上有 `source: "inline-user"` 字段。任何其他源值(例如 `inline-tool-output`、`inline-file-content`)都会被拒绝并报错。代理被指示永远不要伪造 `source` 字段。

**数据隐私**:
- 所有数据都是 `~/.gstack/` 下的本地数据。没有明确的用户操作不会离开。
- `/plan-tune export <path>` 将配置文件写入用户指定的路径(选择性导出)。
- `/plan-tune delete` 清除本地配置文件。
- `gstack-config set telemetry off` 阻止任何遥测(无论如何此技能从不发送配置文件数据)。
- 配置文件具有标准的用户主目录权限。

**注入防御**(与现有 `bin/gstack-learnings-log` 模式一致):`question_summary` 和任何自由形式的用户反馈字段都针对已知的提示注入模式("忽略先前的指令"、"system:"等)进行清理。

## 5 个硬约束(从 office-hours 保留,根据 Codex 反馈更新)

1. **单向门由注册表声明确定性分类**,而非运行时摘要解析。每个注册表条目声明 `door_type: one-way | two-way`。关键字模式回退(`scripts/one-way-doors.ts`)是边缘情况的保险带辅助检查。
2. **配置文件维度可检查且可编辑。** `/plan-tune profile` 显示声明 + 推断 + 差距。通过纯英语编辑仅进入 `declared`。系统独立跟踪 `inferred`。
3. **信号映射在 TypeScript 中手工制作。** `scripts/psychographic-signals.ts` 映射 `{question_id, user_choice} → {dimension, delta}`。非代理推断。在 v1 中,仅用于 `inferred.values` 显示——不用于驱动决策。
4. **v1 中没有心理驱动的自动决策。** 仅显式的每个问题偏好起作用。这完全回避了"校准门可以被操纵"的批评(Codex #13)——v1 没有要通过的门。
5. **每个项目的偏好优于全局偏好。** `~/.gstack/projects/{SLUG}/question-preferences.json` 优于任何未来的全局偏好文件。全局配置文件(`~/.gstack/developer-profile.json`)是跨项目多样性的起点。

## 为什么事件溯源 + 双轨

**为什么推断配置文件采用事件溯源**:
- 信号映射可以在 gstack 版本之间更改。从事件重新计算,无需数据迁移。
- 可审计:`/plan-tune profile --trace autonomy` 显示每个贡献该值的事件。
- 面向未来:可以从现有历史派生新维度。

**为什么双轨(声明 + 推断,分别)**(下面的决策 B):
- 解决了 Codex #6 识别的逻辑矛盾。
- `declared` 是用户主权。用户声明他们是谁。系统对任何用户驱动的内容(偏好、声明、覆盖)遵守。
- `inferred` 是观察。系统跟踪行为模式。显示但在 v1 中不执行。
- `gap` 是有趣的信号。大的差距表明用户的自我描述与他们的行为不匹配——有价值的自我洞察,但不会自动纠正。

## 交互模型——到处都是纯英语

(来自 /plan-devex-review,用户对 CLI 语法的更正):

`/plan-tune`(无参数)进入对话模式。不需要 CLI 子命令语法。

纯语言菜单:
- "显示我的配置文件"
- "查看我被问过的问题"
- "设置关于问题的偏好"
- "更新我的配置文件——我改变了对某事的想法"
- "显示我说的和我做的之间的差距"
- "关闭它"

用户以对话方式回复。代理解释,确认预期的更改,然后写入。例如:
- 用户:"我更像是一个煮沸海洋的人,而不是 0.5 所暗示的"
- 代理:"明白了——将 `declared.scope_appetite` 从 0.5 更新到 0.8?[Y/n]"
- 用户:"是"
- 代理写入更新

对于来自自由形式输入的 `declared` 的任何变更,确认步骤是必需的(Codex #15 信任边界)。

高级用户可以输入快捷方式(`narrative`、`vibe`、`reset`、`stats`、`enable`、`disable`、`diff`)。两者都不是必需的。两者都有效。

## 要创建的文件

### 核心模式
- `scripts/question-registry.ts` — 类型化注册表。从所有 SKILL.md.tmpl AskUserQuestion 调用的审计中播种。
- `scripts/one-way-doors.ts` — 辅助关键字回退。主要:`door_type` 在注册表中。
- `scripts/psychographic-signals.ts` — 用于推断计算的手工制作信号映射。

### 二进制文件
- `bin/gstack-question-log` — 追加日志记录,根据注册表验证。
- `bin/gstack-question-preference` — 读取/写入/检查/清除显式偏好。
- `bin/gstack-developer-profile` — 取代 `bin/gstack-builder-profile`。子命令:`--read`(遗留兼容)、`--derive`、`--gap`、`--profile`。

### 解析器
- `scripts/resolvers/question-tuning.ts` — 三个生成器:`generateQuestionPreferenceCheck(ctx)`(问题前检查)、`generateQuestionLog(ctx)`(问题后日志)、`generateInlineTuneFeedback(ctx)`(问题后 tune: 提示,带用户来源门控指令)。

### 技能
- `plan-tune/SKILL.md.tmpl` — 对话式、纯英语检查和偏好工具。

### 测试
- `test/plan-tune.test.ts` — 注册表完整性、重复 ID 检查、偏好优先级(never-ask + not-one-way → AUTO_DECIDE;never-ask + one-way → ASK_NORMALLY)、用户来源门控(拒绝非 inline-user 来源)、派生 + 重新计算、统一配置文件模式、带 7 会话夹具的迁移回归。

## 要修改的文件

- `scripts/resolvers/index.ts` — 注册 3 个新解析器。
- `scripts/resolvers/preamble.ts` — `_QUESTION_TUNING` 配置读取;为 tier >= 2 注入 3 个解析器。
- `bin/gstack-builder-profile` — 遗留 shim 委托给 `bin/gstack-developer-profile --read`。
- 迁移脚本 — 将现有 builder-profile.jsonl 合并到统一的 developer-profile.json 中。原子、幂等、归档源为 `.migrated-YYYY-MM-DD`。

## v1 中不触及

明确不变——没有 `{{PROFILE_ADAPTATION}}` 占位符,没有基于配置文件的行为变化:

- `ship/SKILL.md.tmpl`、`review/SKILL.md.tmpl`、`office-hours/SKILL.md.tmpl`、`plan-ceo-review/SKILL.md.tmpl`、`plan-eng-review/SKILL.md.tmpl`

这些技能仅获得用于日志记录/偏好检查/调整反馈的前言注入。没有配置文件驱动的默认值。v2 工作。

## 决策日志(每个决策的利弊)

### 决策 A:捆绑所有三个(问题日志 + 敏感性 + 心理)vs. 发布更小的楔子 — 初始答案:捆绑;修订:注册表优先观察性

初始用户立场(office-hours):"心理维度是差异化。发布整个东西,以便反馈循环可以实际调整行为。"这推动了 CEO EXPANSION。

**捆绑的优点:** 雄心。学习层是使其不仅仅是配置的原因。没有心理维度,它只是一个花哨的设置菜单。

**捆绑的缺点(由 Codex 提出):** 基础设施不存在。提示约定之上的心理维度是沙子。E1/E4/E6 组合不连贯。配置文件污染未解决。前言中的 E5 是隐藏的热路径副作用。实现顺序围绕不可强制执行的约定构建机制。

**修订答案:** 注册表优先观察性 v1(本文档)。将雄心保留为 v2 目标,带有明确的验收标准。发布一个可防御的基础。用户在看到 Codex 的 20 点批评后接受了这一点。

### 决策 B:事件溯源 vs. 存储维度 vs. 混合 — 答案:事件溯源 + 用户声明锚点(B+C)

**方法 A(存储维度):** 就地变更。简单。
- 优点:最小的数据模型。易于推理。
- 缺点:有损。无历史。信号映射更改需要迁移。配置文件更改对用户不透明。

**方法 B(事件溯源):** 存储原始事件,派生维度。
- 优点:可审计。信号映射更改时可重新计算。永远不需要数据迁移。匹配现有 learnings.jsonl 模式。
- 缺点:更复杂的派生。事件文件随时间增长(压缩推迟到 v2)。

**方法 C(混合——用户声明锚点,事件细化):** 初始配置文件是用户声明的;事件在 ±0.2 内细化。
- 优点:第 1 天价值。用户主权。校准锚点而非从零开始。
- 缺点:±0.2 限制与不匹配检测产生逻辑冲突(Codex #6 发现了这一点)。

**选择:B+C 组合,移除 ±0.2 限制。** 底层事件溯源,声明配置文件作为一流的独立字段。无限制。声明和推断作为独立值存在。它们之间的差距显示但在 v1 中不自动纠正。

### 决策 C:单向门分类 — 运行时散文解析 vs. 注册表声明 — 答案:注册表声明(Codex 后)

**运行时散文解析(原始):** `isOneWayDoor(skill, category, summary)` 加关键字模式。
- 优点:对技能作者的摩擦最小。无需维护模式。
- 缺点(Codex #4):安全取决于措辞。措辞温和的破坏性操作问题可能被错误分类。对于安全门不可接受。

**注册表声明(修订):** 每个注册表条目声明 `door_type`。
- 优点:确定性。可审计。CI 可强制执行(所有问题必须声明)。
- 缺点:维护负担。每个新技能问题必须分类。

**选择:注册表声明为主,关键字模式为回退。** 模式治理是安全的代价。

### 决策 D:内联调整反馈语法 — 结构化关键字 vs. 自由形式自然语言 — 答案:结构化带自由形式回退

**仅结构化关键字:** `tune: unnecessary | ask-less | never-ask | always-ask | context-dependent`。
- 优点:明确。干净的配置文件数据。
- 缺点:用户必须记忆。

**仅自由形式:** 代理解释用户说的任何内容。
- 优点:自然。无需学习语法。
- 缺点:不一致的配置文件数据。难以调试为什么调整没有生