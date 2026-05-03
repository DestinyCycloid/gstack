# /guard — 完全安全模式

激活破坏性命令警告和目录范围编辑限制。
这是 `/careful` + `/freeze` 的组合命令。

**依赖说明:** 此技能引用了同级 `/careful` 和 `/freeze` 技能目录中的钩子脚本。两者都必须安装(它们由 gstack 设置脚本一起安装)。

```bash
mkdir -p ~/.gstack/analytics
echo '{"skill":"guard","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
```

## 设置

询问用户要将编辑限制在哪个目录。使用 AskUserQuestion:

- 问题: "守护模式:应将编辑限制在哪个目录?破坏性命令警告始终开启。所选路径之外的文件将被阻止编辑。"
- 文本输入(非多选) — 用户输入路径。

用户提供目录路径后:

1. 将其解析为绝对路径:
```bash
FREEZE_DIR=$(cd "<user-provided-path>" 2>/dev/null && pwd)
echo "$FREEZE_DIR"
```

2. 确保尾部斜杠并保存到冻结状态文件:
```bash
FREEZE_DIR="${FREEZE_DIR%/}/"
eval "$(~/.claude/skills/gstack/bin/gstack-paths)"
STATE_DIR="$GSTACK_STATE_ROOT"
mkdir -p "$STATE_DIR"
echo "$FREEZE_DIR" > "$STATE_DIR/freeze-dir.txt"
echo "Freeze boundary set: $FREEZE_DIR"
```

告知用户:
- "**守护模式已激活。** 现在运行两项保护:"
- "1. **破坏性命令警告** — rm -rf、DROP TABLE、force-push 等将在执行前发出警告(您可以覆盖)"
- "2. **编辑边界** — 文件编辑限制在 `<path>/`。此目录之外的编辑将被阻止。"
- "要移除编辑边界,运行 `/unfreeze`。要停用所有保护,结束会话。"

## 保护内容

有关破坏性命令模式和安全例外的完整列表,请参阅 `/careful`。
有关编辑边界强制执行的工作原理,请参阅 `/freeze`。