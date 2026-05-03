# GCOMPACTION.md — 设计与架构(已搁置)

**批准后的目标路径:** `docs/designs/GCOMPACTION.md`

这是 `gstack compact` 的保留设计文档。下方第一个 `---` 分隔符之上的所有内容在计划批准时会被逐字提取到 `docs/designs/GCOMPACTION.md`。分隔符之后是归档研究(办公时间 + 竞品深度分析 + 工程评审笔记 + codex 评审 + 研究发现),这些内容为设计提供了依据。

---

## 状态:已搁置(2026-04-17)— 等待 Anthropic `updatedBuiltinToolOutput` API

**搁置原因。** v1 架构假设 Claude Code 的 `PostToolUse` 钩子可以替换进入模型上下文的内置工具(Bash、Read、Grep、Glob、WebFetch)的工具输出。2026-04-17 的研究确认这在当前是不可能的。

**证据:**

1. **官方文档**(https://code.claude.com/docs/en/hooks):`PostToolUse` 唯一记录的输出替换字段是 `hookSpecificOutput.updatedMCPToolOutput`,文档明确指出:*"仅适用于 MCP 工具:用提供的值替换工具的输出。"* 内置工具不存在等效字段。
2. **Anthropic issue [#36843](https://github.com/anthropics/claude-code/issues/36843)**(未解决):Anthropic 自己承认了这个缺口。*"PostToolUse 钩子可以通过 `updatedMCPToolOutput` 替换 MCP 工具输出,但内置工具(WebFetch、WebSearch、Bash、Read 等)没有等效功能...它们只能通过 `decision: block`(注入原因字符串)或 `additionalContext` 添加警告。原始恶意内容仍会到达模型。"*
3. **RTK 机制**(源码审查位于 `src/hooks/init.rs:906-912` 和 `hooks/claude/rtk-rewrite.sh:83-100`):RTK 不是 PostToolUse 压缩器。它是一个 **PreToolUse** Bash 匹配器,重写 `tool_input.command`(例如 `git status` → `rtk git status`)。包装的命令本身产生紧凑的 stdout。RTK README 确认:*"钩子仅在 Bash 工具调用上运行。Claude Code 内置工具如 Read、Grep 和 Glob 不通过 Bash 钩子,因此不会自动重写。"* RTK 仅限 Bash 是架构约束,而非选择。
4. **tokenjuice 机制**(源码审查位于 `src/core/claude-code.ts:160, 491, 540-549`):tokenjuice 确实用 `matcher: "Bash"` 注册了 `PostToolUse`,但没有真正的输出替换 API 可用 — 它劫持 `decision: "block"` + `reason` 来注入压缩文本。这是否真正减少了模型上下文令牌还是只是覆盖了 UI 输出存在争议。tokenjuice 也仅限 Bash。
5. **Read/Grep/Glob 在 Claude Code 内部进程中执行**,完全绕过钩子。楔子(ii)"原生工具覆盖"从一开始就在架构上不可能,无论是否有替换 API。

**后果。** 两个楔子在其原始形式下都已失效:
- 楔子(i)"条件 LLM 验证器" — 技术上仍然可行,但仅适用于 Bash 输出,通过 PreToolUse 命令包装(RTK 的机制)。一旦我们也仅限 Bash,验证器就不再是差异化因素。
- 楔子(ii)"原生工具覆盖" — 目前不可能。Read/Grep/Glob 不触发钩子。即使触发,也不存在输出替换字段。

**决定。** 完全搁置 `gstack compact`。跟踪 Anthropic issue #36843 以获取 `updatedBuiltinToolOutput`(或等效功能)的到来。当该 API 发布时,本设计文档 + 下方锁定的 15 个决策 + 底部的研究档案将成为新实施冲刺的解锁文档。

**如果取消搁置:** 从下方"计划工程评审期间锁定的决策"块开始 — 大多数仍然有效。然后根据新发布的 API 重新验证钩子参考,更新架构数据流图以使用任何真实的输出替换字段,并在编码前对修订后的计划重新运行 `/codex review`。

**我们不做的事:**
- 不发布仅限 Bash 的 PreToolUse 包装器。那是 RTK 的产品;他们有 28K 星和 3 年的规则伤疤。没有楔子。
- 不发布 `decision: block` + `reason` hack。未记录的行为,Anthropic 可能会破坏它,模型可能仍会在压缩覆盖旁边看到原始输出 — 上下文节省存在争议。
- 不单独发布 B 系列基准测试。没有工作的压缩器,就没有什么可基准测试的。

**搁置成本:** ~0。没有编写代码。设计文档 + 研究 + 决策作为随时可解锁的文档保留。

---

## 计划工程评审期间锁定的决策(2026-04-17)

为取消搁置冲刺保留,如果/当 Anthropic 发布内置工具输出替换 API 时。

工程评审期间做出的每个决策的摘要。完整理由保留在下方各节中;如果其他内容有偏差,此块是唯一的真实来源。

**范围(第 0 节):**
1. **Claude 优先 v1。** 仅在 Claude Code 上发布 compact + 规则 + 验证器。Codex + OpenClaw 在主要主机上证明楔子后在 v1.1 登陆。减少约 2 天的主机集成并降低发布风险。原始"楔子(ii)原生工具覆盖"声明适用于 v1 的 Claude Code;在 v1.1 之前我们不做跨主机声明。
2. **13 规则启动库。** v1 发布测试(jest/vitest/pytest/cargo-test/go-test/rspec)+ git(diff/log/status)+ 安装(npm/pnpm/pip/cargo)。构建/lint/日志系列推迟到 v1.1,由真实用户的 `gstack compact discover` 遥测驱动。
3. **验证器在 v1.0 默认开启。** `failureCompaction` 触发器(exit≠0 且 >50% 减少)开箱即用。验证器是楔子 — 默认关闭会隐藏差异化功能。触发器边界已将预期触发率保持在工具调用的 ≤10%。

**架构(第 1 节):**
4. **Haiku 输出的精确行匹配清理。** 按 `\n` 分割原始输出,将行放入集合中,仅追加 Haiku 中逐字出现在该集合中的行。最严格的对抗性契约;提示注入尝试无法插入新文本。
5. **分层 failureCompaction 信号。** 优先使用信封中的 `exitCode`;如果主机省略它,则回退到输出上的 `/FAIL|Error|Traceback|panic/` 正则表达式。在 `meta.failureSignal`("exit" | "pattern" | "none")中记录触发的信号。预实施任务 #1 仍会凭经验验证 Claude Code 的信封,但如果没有,系统不再崩溃。
6. **深度合并规则解析。** 用户/项目规则继承它们未覆盖的内置字段。逃生舱口:规则文件中的 `"extends": null` 触发完全替换语义。匹配 eslint/tsconfig/.gitignore 的心智模型 — 覆盖一部分而不丢失其余部分。

**代码质量(第 2 节):**
7. **每规则正则表达式超时,无 RE2 依赖。** 通过 50ms AbortSignal 预算运行每个规则的正则表达式;超时时,跳过规则并记录 `meta.regexTimedOut: [ruleId]`。避免 WASM 依赖并保持规则作者语法不受约束。
8. **预编译规则包。** `gstack compact install` 和 `gstack compact reload` 生成 `~/.gstack/compact/rules.bundle.json`(深度合并,正则表达式编译的元数据已缓存)。钩子读取该单个文件而不是解析 N 个源文件。
9. **mtime 漂移时自动重新加载。** 钩子在启动时统计规则源文件;如果任何源文件比包新,则在应用前内联重建。每次调用增加约 0.5ms,但消除了"我编辑了规则但什么都没改变"的陷阱。
10. **扩展的 v1 编辑集。** Tee 文件编辑:AWS 密钥、GitHub 令牌(`ghp_/gho_/ghs_/ghu_`)、GitLab 令牌(`glpat-`)、Slack webhooks、通用 JWT(三个 base64 段)、通用 bearer 令牌、SSH 私钥头(`-----BEGIN * PRIVATE KEY-----`)。信用卡/SSN/每键环境对推迟到 v2 的完整 DLP 层。

**测试(第 3 节):**
11. **P 系列门控子集。** v1 门控层 P 测试:P1(二进制垃圾)、P3(空输出)、P6(RTK 杀手关键堆栈帧)、P8(tee 的秘密)、P15(钩子超时)、P18(提示注入)、P26(格式错误的用户规则 JSON)、P28(正则表达式 DoS)、P30(Haiku 幻觉)。剩余 21 个 P 案例随着真实错误的出现增长 R 系列。
12. **固件版本标记。** 每个黄金固件都有 `toolVersion:` 前言。当固件 toolVersion ≠ 当前安装时,CI 发出警告。不再基于日历的轮换。
13. **B 系列真实世界基准测试台(硬 v1 门控)。** 新组件 `compact/benchmark/` 扫描 `~/.claude/projects/**/*.jsonl`,对最嘈杂的工具调用进行排名,将它们聚类为命名场景,针对它们重放压缩器,并按规则系列报告减少。v1 在作者自己的 30 天语料库上的 B 系列显示 ≥15% 减少且在植入的错误上零关键行丢失之前无法发布。仅本地;从不上传。社区共享语料库是 v2。

**性能(第 4 节):**
14. **修订的延迟预算。** macOS ARM 上的 Bun 冷启动为 15-25ms;原始 10ms p50 目标不现实。新预算:macOS ARM 上 <30ms p50 / <80ms p99,Linux 上 <20ms p50 / <60ms p99(验证器关闭)。验证器触发预算保持 <600ms p50 / <2s p99。守护进程模式是 v2 选项,取决于 B 系列显示冷启动损害会话节省。
15. **面向行的流式管道。** 通过 stdin 读取行 → 过滤 → 分组 → 去重 → 环形缓冲尾部截断 → stdout。任何单行 >1MB 触发 P9(截断到 1KB,带 `[... truncated ...]` 标记)。无论总输出大小如何,内存上限为 64MB。

上面的每一行都是实施中的 `MUST`。偏差需要新的工程评审。

---

## 摘要

`gstack compact` 被设计为一个 `PostToolUse` 钩子,在工具输出噪声到达 AI 编码代理的上下文窗口之前减少它。确定性 JSON 规则将缩小嘈杂的测试运行器、构建日志、git diff 和包安装。条件 Claude Haiku 验证器将在过度压缩风险高时充当安全网。

**当前状态:已搁置。** 见上方"状态"部分。该架构依赖于截至 2026-04-17 不存在的 Claude Code API(`updatedBuiltinToolOutput` 或内置工具的等效功能)。Anthropic issue #36843 跟踪该缺口。

**预期目标(为取消搁置冲刺保留):** 每个长会话 15-30% 的工具输出令牌减少,任务失败率零增加。

**原始楔子(相对于 28K 星现任者 RTK)— 两者都被研究无效:**
1. ~~**条件 LLM 验证器。**~~ 通过 PreToolUse 命令包装在技术上仍然可行,但仅适用于 Bash。一旦我们仅限 Bash,就不再是差异化因素。如果内置工具 API 到来,请重新考虑。
2. ~~**原生工具覆盖。**~~ 目前在架构上不可能。Read/Grep/Glob 在 Claude Code 内部进程中执行,不触发钩子。即使对于触发 `PostToolUse` 的工具,非 MCP 工具也不存在输出替换字段。

**原始定位(现已无效):** *"RTK 很快。gstack compact 既快又安全,它覆盖工具箱中的每个工具,而不仅仅是 Bash。"*

## 非目标

- 总结用户消息或先前的代理回合(Claude 自己的 Compaction API 拥有该功能)。
- 压缩代理响应输出(caveman 的层)。
- 缓存工具调用以避免重新执行(token-optimizer-mcp 的层)。
- 充当通用日志分析器。
- 替换代理自己关于何时使用 `GSTACK_RAW=1` 重新运行命令的判断。

## 为什么值得构建

**问题是可测量的,而非假设的。**

- [Chroma 研究(2025)](https://research.trychroma.com/context-rot)测试了 18 个前沿模型。每个模型随着上下文增长而退化。腐烂在窗口限制之前就开始了 — 200K 模型在 50K 时腐烂。
- 编码代理是最坏的情况:累积上下文 + 高干扰密度 + 长任务范围。工具输出被明确命名为主要噪声源。
- 市场已投票:Anthropic 发布了 Opus 4.6 Compaction API;OpenAI 发布了压缩指南;Google ADK 发布了上下文压缩;LangChain 发布了自主压缩;sst/opencode 具有内置压缩。混合确定性 + LLM 模式是行业共识。

**现有领域(gstack compact 加入并与之区分的):**

| 项目 | 星标 | 许可证 | 层 | 威胁 | 注释 |
|---------|-------|---------|-------|--------|------|
| **RTK (rtk-ai/rtk)** | **28K** | Apache-2.0 | 工具输出 | 主要基准 | 纯 Rust,仅 Bash,零 LLM |
| caveman | 34.8K | MIT | 输出令牌 | 不同轴 | 简洁系统提示;与我们配对 |
| claude-token-efficient | 4.3K | MIT | 响应冗长 | 不同轴 | 单个 CLAUDE.md |
| token-optimizer-mcp | 49 | MIT | MCP 缓存 | 不同轴 | 防止调用而不是压缩输出 |
| tokenjuice | ~12 | MIT | 工具输出 | 太新 | 2 天前;启发了我们的 JSON 信封 |
| 6 层令牌节省堆栈 | — | 公共 gist | 配方 | 零 | 文档;验证堆叠压缩论文 |

RTK 是唯一的直接竞争对手。其他所有内容都压缩不同的令牌源。

**许可证兼容性:** 每个引用的项目都是宽松许可的(MIT 或 Apache-2.0),与 gstack 的 MIT 许可证兼容。没有 AGPL、GPL 或其他 copyleft 依赖项。有关洁净室政策,请参阅下方"许可证和归属"部分。

## 架构

### 数据流

```
┌─────────────────────────────────────────────────────────────────┐
│  主机(Claude Code / Codex / OpenClaw)                           │
│  ─────────────────────────────────────────                      │
│  1. 代理请求工具调用:Bash|Read|Grep|Glob|MCP                    │
│  2. 主机执行工具                                                 │
│  3. 主机使用以下内容调用 PostToolUse 钩子:{tool, input, output} │
└────────────────────┬────────────────────────────────────────────┘
                     │ stdin(JSON 信封)
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│  gstack-compact 钩子二进制文件                                   │
│  ───────────────────────────                                    │
│  a. 解析信封                                                     │
│  b. 按(tool, command, pattern)匹配规则                          │
│  c. 应用规则原语:filter / group / truncate / dedupe             │
│  d. 记录减少元数据                                               │
│  e. 评估验证器触发器                                             │
│  f. 如果触发器满足:调用 Haiku,追加保留的行                      │
│  g. 失败退出代码时:将原始内容 tee 到 ~/.gstack/compact/tee/...  │
│  h. 向 stdout 发出 JSON 信封                                     │
└────────────────────┬────────────────────────────────────────────┘
                     │ stdout(JSON 信封)
                     ▼
              主机将压缩输出替换到代理上下文中
```

### 规则解析

三层层次结构(最高优先级获胜),与 tokenjuice 和 gstack 现有的主机配置导出模型相同的模式:

1. 内置规则:随 gstack 一起发布的 `compact/rules/`
2. 用户规则:`~/.config/gstack/compact-rules/`
3. 项目规则:`.gstack/compact-rules/`

规则按规则 ID 匹配工具调用。ID 为 `tests/jest` 的项目规则完全覆盖内置的 `tests/jest`。没有合并 — 替换语义,以保持推理简单。

### JSON 信封契约(采用自 tokenjuice)

输入:
```json
{
  "tool": "Bash",
  "command": "bun test test/billing.test.ts",
  "argv": ["bun", "test", "test/billing.test.ts"],
  "combinedText": "...",
  "exitCode": 1,
  "cwd": "/Users/garry/proj",
  "host": "claude-code"
}
```

输出:
```json
{
  "reduced": "带有 [gstack-compact: N → M lines, rule: X] 头的压缩输出",
  "meta": {
    "rule": "tests/jest",
    "linesBefore": 247,
    "linesAfter": 18,
    "bytesBefore": 18234,
    "bytesAfter": 892,
    "verifierFired": false,
    "teeFile": null,
    "durationMs": 8
  }
}
```

### 规则模式

紧凑,最小。总规则有效负载必须在磁盘上保持 <5KB(来自 claude-token-efficient 的教训:规则文件本身在每个会话中消耗令牌)。

```json
{
  "id": "tests/jest",
  "family": "test-results",
  "description": "Jest/Vitest 输出 — 保留失败和摘要计数",
  "match": {
    "tools": ["Bash"],
    "commands": ["jest", "vitest", "bun test"],
    "patterns": ["jest", "vitest", "PASS", "FAIL"]
  },
  "primitives": {
    "filter": {
      "strip": ["\\x1b\\[[0-9;]*m", "^\\s*at .+node_modules"],
      "keep": ["FAIL", "PASS", "Error:", "Expected:", "Received:", "✓", "✗", "Tests:"]
    },
    "group": {
      "by": "error-kind",
      "header": "按类型分组的错误:"
    },
    "truncate": {
      "headLines": 5,
      "tailLines": 15,
      "onFailure": { "headLines": 20, "tailLines": 30 }
    },
    "dedupe": {
      "pattern": "^\\s*$",
      "format": "[... {count} 个空行 ...]"
    }
  },
  "tee": {
    "onExit": "nonzero",
    "maxBytes": 1048576
  },
  "counters": [
    { "name": "failed", "pattern": "^FAIL\\s", "flags": "m" },
    { "name": "passed", "pattern": "^PASS\\s", "flags": "m" }
  ]
}
```

四个原语 — `filter`、`group`、`truncate`、`dedupe` — 直接从 RTK 的技术分类法中提取(每个严肃的压缩器都需要处理的唯一内容)。任何规则都可以组合四个原语的任何子集;省略的原语是无操作。

### 验证器层(分层,选择加入)

验证器是一个廉价的 Haiku 调用,仅在特定触发器下触发。从不在每次工具调用时触发。

**触发器矩阵(用户可配置):**

| 触发器 | 默认 | 条件 |
|---------|---------|-----------|
| `failureCompaction` | **开启** | 退出代码 ≠ 0 且减少 >50%(诊断有风险)|
| `aggressiveReduction` | 关闭 | 减少 >80% 且原始 >200 行 |
| `largeNoMatch` | 关闭 | 没有规则匹配且输出 >500 行 |
| `userOptIn` | 开启(环境门控)| `GSTACK_COMPACT_VERIFY=1` 强制该调用的验证器 |

默认配置仅发布 `failureCompaction` — 最高杠杆情况(代理正在调试;规则可能已过滤关键堆栈帧)。

**Haiku 的工作(有界):**

```
这是原始输出(截断到前 2000 行)和压缩版本。
返回压缩版本中缺少的原始输出中的任何重要行,
或者如果没有缺少关键内容则返回 `NONE`。
```

验证器从不重写压缩输出。它只在标题下追加缺失的行:

```
[gstack-compact: 247 → 18 lines, rule: tests/jest]
[gstack-verify: Haiku 保留的 2 个额外行]
  TypeError: Cannot read property 'foo' of undefined
    at parseConfig (src/config.ts:42:18)
```

**为什么是 Haiku,而不是 Sonnet:** 成本约为 1/12,约 500ms 对比约 2s,任务是简单的子字符串分类,而不是推理。

**验证器配置(`compact/rules/_verifier.json`):**

```json
{
  "verifier": {
    "enabled": true,
    "model": "claude-haiku-4-5-20251001",
    "maxInputLines": 2000,
    "triggers": {
      "aggressiveReduction": { "enabled": false, "thresholdPct": 80, "minLines": 200 },
      "failureCompaction":   { "enabled": true,  "minReductionPct": 50 },
      "largeNoMatch":        { "enabled": false, "minLines": 500 },
      "userOptIn":           { "enabled": true, "envVar": "GSTACK_COMPACT_VERIFY" }
    },
    "fallback": "passthrough"
  }
}
```

**失败模式(验证器严格是附加的 — 从不破坏基线):**

- 没有 `ANTHROPIC_API_KEY` → 跳过验证器,使用纯规则输出。
- Haiku 调用超时(>5s)→ 跳过验证器,使用纯规则输出。
- Haiku 返回格式错误的 JSON → 跳过,使用纯规则输出。
- Haiku 返回提示注入尝试 → 清理:仅追加原始原始输出的子字符串匹配的行。
- Haiku 返回幻觉行(原始中不存在)→ 丢弃它们。

### Tee 模式(采用自 RTK)

对于任何退出代码 ≠ 0 的命令,完整的未过滤输出将写入 `~/.gstack/compact/tee/{timestamp}_{cmd-slug}.log`。压缩输出包括 tee 文件指针:

```
[gstack-compact: 247 → 18 lines, rule: tests/jest, tee: ~/.gstack/compact/tee/20260416-143022_bun-test.log]
```

如果代理需要完整的堆栈跟踪,可以直接读取 tee 文件。这用更清晰的设计替换了早期的 `onFailure.preserveFull` 机制:压缩输出始终保持小;原始输出始终只需一个 `cat`。

**Tee 安全性:**

- 文件模式 `0600` — 非全局可读。
- 内置秘密正则表达式集在写入前编辑 AWS 密钥、bearer 令牌和常见凭据模式。
- 失败的写入(只读文件系统,权限被拒绝)优雅降级:仍发出压缩输出,记录 `meta.teeFailed: true`。
- Tee 文件在 7 天后自动过期(钩子启动时清理)。

### 主机集成矩阵

| 主机 | 钩子类型 | 支持的匹配器 | 配置路径 |
|------|-----------|-------------------|-------------|
| Claude Code | `PostToolUse` | Bash, Read, Grep, Glob, Edit, Write, WebFetch, WebSearch, mcp__* | `~/.claude/settings.json` |
| Codex (v1.1) | `PostToolUse` 等效 | Bash(主要);工具子集待定 — 经验验证是 v1.1 先决条件 | `~/.codex/hooks.json` |
| OpenClaw (v1.1) | 原生