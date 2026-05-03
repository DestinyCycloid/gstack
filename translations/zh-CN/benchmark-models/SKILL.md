# /benchmark-models — 跨模型技能基准测试

你正在运行 `/benchmark-models` 工作流。使用交互式流程包装 `gstack-model-benchmark` 二进制文件,选择提示词、确认提供商、预览认证并运行基准测试。

与 `/benchmark` 不同 — 该技能测量网页性能(核心 Web 指标、加载时间)。本技能测量 AI 模型在 gstack 技能或任意提示词上的性能。

---

## 步骤 0: 定位二进制文件

```bash
BIN="$HOME/.claude/skills/gstack/bin/gstack-model-benchmark"
[ -x "$BIN" ] || BIN=".claude/skills/gstack/bin/gstack-model-benchmark"
[ -x "$BIN" ] || { echo "ERROR: gstack-model-benchmark not found. Run ./setup in the gstack install dir." >&2; exit 1; }
echo "BIN: $BIN"
```

如果未找到,停止并告知用户重新安装 gstack。

---

## 步骤 1: 选择提示词

使用 AskUserQuestion,采用前导格式:
- **重新定位:** 当前项目 + 分支。
- **简化:** "跨模型基准测试会在 2-3 个 AI 模型上运行相同的提示词,向你展示它们在速度、成本和输出质量上的比较。我们应该使用什么提示词?"
- **推荐:** A,因为针对真实技能进行基准测试会暴露工具使用差异,而不仅仅是原始生成能力。
- **选项:**
  - A) 对我的某个 gstack 技能进行基准测试(我们接下来会选择哪个技能)。完整度: 10/10。
  - B) 使用内联提示词 — 在下一轮输入。完整度: 8/10。
  - C) 指向磁盘上的提示词文件 — 在下一轮指定路径。完整度: 8/10。

如果选 A: 列出具有 SKILL.md 文件的顶级 gstack 技能(从 `find . -maxdepth 2 -name SKILL.md -not -path './.*'` 获取),通过第二个 AskUserQuestion 让用户选择一个。使用选中的 SKILL.md 路径作为提示词文件。

如果选 B: 询问用户内联提示词。通过 `--prompt "<text>"` 原样使用。

如果选 C: 询问路径。验证文件存在。用作位置参数。

---

## 步骤 2: 选择提供商

```bash
"$BIN" --prompt "unused, dry-run" --models claude,gpt,gemini --dry-run
```

显示试运行输出。"Adapter availability" 部分告诉用户哪些提供商会实际运行(OK)与跳过(NOT READY — 包含修复提示)。

如果所有三个都显示 NOT READY: 停止并给出清晰消息 — 基准测试至少需要一个已认证的提供商才能运行。建议运行 `claude login`、`codex login` 或 `gemini login` / `export GOOGLE_API_KEY`。

如果至少有一个是 OK: AskUserQuestion:
- **简化:** "我们应该包含哪些模型?上面的试运行显示了哪些已认证。未认证的会被干净地跳过 — 不会中止批处理。"
- **推荐:** A(所有已认证的提供商),因为运行尽可能多的提供商能提供最丰富的比较。
- **选项:**
  - A) 所有已认证的提供商。完整度: 10/10。
  - B) 仅 Claude。完整度: 6/10(无跨模型信号 — 对于单独的 claude 基准测试,使用 /ship 的审查)。
  - C) 选择两个 — 在下一轮指定。完整度: 8/10。

---

## 步骤 3: 决定是否使用评判器

```bash
[ -n "$ANTHROPIC_API_KEY" ] || grep -q 'ANTHROPIC' "$HOME/.claude/.credentials.json" 2>/dev/null && echo "JUDGE_AVAILABLE" || echo "JUDGE_UNAVAILABLE"
```

如果评判器可用,AskUserQuestion:
- **简化:** "质量评判器使用 Anthropic 的 Claude 作为决胜局,对每个模型的输出进行 0-10 分评分。每次运行增加约 $0.05。如果你关心输出质量而不仅仅是延迟和成本,建议启用。"
- **推荐:** A — 重点就是比较质量,而不仅仅是速度。
- **选项:**
  - A) 启用评判器(增加约 $0.05)。完整度: 10/10。
  - B) 跳过评判器 — 仅速度/成本/令牌数。完整度: 7/10。

如果评判器不可用,跳过此问题并省略 `--judge` 标志。

---

## 步骤 4: 运行基准测试

根据步骤 1、2、3 的决定构建命令:

```bash
"$BIN" <prompt-spec> --models <picked-models> [--judge] --output table
```

其中 `<prompt-spec>` 是 `--prompt "<text>"`(步骤 1B)或文件路径(步骤 1A 或 1C),`<picked-models>` 是步骤 2 中的逗号分隔列表。

流式输出结果。这个过程很慢 — 每个提供商都会完整运行提示词。根据提示词复杂度和是否启用 `--judge`,预计需要 30 秒到 5 分钟。

---

## 步骤 5: 解读结果

表格打印后,为用户总结:
- **最快** — 延迟最低的提供商。
- **最便宜** — 成本最低的提供商。
- **最高质量**(如果运行了 `--judge`) — 得分最高的提供商。
- **综合最佳** — 使用判断。如果运行了评判器:质量加权。否则:指出用户需要做出的权衡。

如果任何提供商遇到错误(认证/超时/速率限制),指出并提供修复路径。

---

## 步骤 6: 提供保存结果选项

AskUserQuestion:
- **简化:** "将此基准测试保存为 JSON,以便将来的运行与之比较?"
- **推荐:** A — 随着提供商更新模型,技能性能会漂移;保存的基线可以捕获质量退化。
- **选项:**
  - A) 保存到 `~/.gstack/benchmarks/<date>-<skill-or-prompt-slug>.json`。完整度: 10/10。
  - B) 仅打印,不保存。完整度: 5/10(丢失趋势数据)。

如果选 A: 使用 `--output json` 重新运行并 tee 到带日期的文件。打印路径,以便用户可以将未来的运行与之对比。

---

## 重要规则

- **在步骤 2 的试运行之前,绝不运行真实基准测试。** 用户需要在花费 API 调用之前看到认证状态。
- **绝不硬编码模型名称。** 始终从用户的步骤 2 选择传递提供商 — 二进制文件处理其余部分。
- **绝不自动包含 `--judge`。** 它会增加实际成本;用户必须选择加入。
- **如果零个提供商已认证,停止。** 不要尝试基准测试 — 它不会产生有用的输出。
- **成本是可见的。** 每次运行都会在表格中显示每个提供商的成本。用户应该在下次运行前看到它。