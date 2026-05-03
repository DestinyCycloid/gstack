# freeze

将文件编辑限制在特定目录中。阻止在允许路径之外使用 Edit 和 Write 工具。在调试时使用此功能可防止意外"修复"无关代码,或者当你想将更改范围限定在一个模块时使用。当被要求"冻结"、"限制编辑"、"仅编辑此文件夹"或"锁定编辑"时使用。

## 设置

询问用户要将编辑限制在哪个目录。使用 AskUserQuestion:

- 问题:"我应该将编辑限制在哪个目录?此路径之外的文件将被阻止编辑。"
- 文本输入(不是多项选择)——用户输入路径。

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

告诉用户:"编辑现已限制在 `<path>/`。任何在此目录之外的 Edit 或 Write 操作都将被阻止。要更改边界,请再次运行 `/freeze`。要移除限制,请运行 `/unfreeze` 或结束会话。"

## 工作原理

钩子从 Edit/Write 工具输入 JSON 中读取 `file_path`,然后检查路径是否以冻结目录开头。如果不是,则返回 `permissionDecision: "deny"` 以阻止操作。

冻结边界通过状态文件在会话期间持久保存。钩子脚本在每次 Edit/Write 调用时读取它。

## 注意事项

- 冻结目录上的尾部 `/` 可防止 `/src` 匹配 `/src-old`
- 冻结仅适用于 Edit 和 Write 工具——Read、Bash、Glob、Grep 不受影响
- 这可以防止意外编辑,但不是安全边界——像 `sed` 这样的 Bash 命令仍然可以修改边界之外的文件
- 要停用,请运行 `/unfreeze` 或结束对话