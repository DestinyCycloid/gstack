# 领域技能

代理为自己编写的每个站点笔记。跨会话累积:一旦代理发现了关于某个网站的非显而易见的内容,它就会保存一个技能,该主机上的未来会话会将该笔记注入到它们的提示上下文中。

这是 gstack 从 [browser-use/browser-harness](https://github.com/browser-use/browser-harness) 借鉴的功能。
gstack 复制了每个站点笔记的模式,而不是自修改运行时模式。技能是加载到提示中的 markdown 文本;它们不是可执行代码。

## 代理如何使用

```bash
# 代理在成功完成任务后记录了它对站点的了解。
# 主机名自动从活动标签页获取(无需代理参数)。
echo "# LinkedIn Apply Button

The Apply button on /jobs/view pages is inside an iframe with a class
matching 'jobs-apply-button-iframe'. Use \$B frame --url 'apply' first,
then snapshot." | $B domain-skill save

# 查看已保存的内容
$B domain-skill list

# 读取特定主机技能的正文
$B domain-skill show linkedin.com

# 在 $EDITOR 中交互式编辑
$B domain-skill edit linkedin.com

# 将活动的每个项目技能提升为全局(跨项目)
$B domain-skill promote-to-global linkedin.com

# 回滚最近的编辑
$B domain-skill rollback linkedin.com

# 删除(墓碑标记 — 可通过回滚恢复)
$B domain-skill rm linkedin.com
```

## 状态机

```
  ┌──────────────┐  3 次成功使用              ┌────────┐  promote-to-global   ┌────────┐
  │ quarantined  │ ─────────────────────▶  │ active │ ──────────────────▶  │ global │
  │ (每个项目)    │  (无分类器标记)           │(项目)  │  (手动命令)           │        │
  └──────────────┘                          └────────┘                      └────────┘
         ▲                                       │
         │  使用期间分类器标记                     │  rollback (版本日志)
         └───────────────────────────────────────┘
```

新保存的技能会处于 **quarantined** 状态,不会自动在提示中触发。在该主机上使用 3 次且 L4 ML 分类器未标记技能内容后,该技能会自动提升为项目中的 **active** 状态。活动技能会在该主机名的每个新侧边栏代理会话中触发。

要使技能跨项目触发(例如,"我希望我的 LinkedIn 技能在我工作的每个 gstack 项目中都可用"),请明确运行
`$B domain-skill promote-to-global <host>`。这是按设计选择加入的(Codex T4 外部审查):全面的跨项目累积会在不相关的工作之间泄漏上下文。

## 存储

技能存储在两个位置:

- **每个项目**: `~/.gstack/projects/<slug>/learnings.jsonl` — 与 `/learn` 技能使用的相同 JSONL 文件。领域技能是 `type:"domain"` 行。
- **全局**: `~/.gstack/global-domain-skills.jsonl` — 仅包含 `state:"global"` 行。

两个文件都是仅追加的 JSONL。删除使用墓碑标记;空闲压缩器会定期重写文件。容错解析器在读取时会丢弃部分尾随行,因此写入过程中的崩溃不会破坏后续读取。

## 安全模型

技能是加载到未来提示上下文中的代理编写内容。这使它们成为经典的代理到代理提示注入向量。该计划通过多层明确解决了这个问题:

| 层 | 内容 | 位置 |
|-------|------|-------|
| L1-L3 | 数据标记、隐藏元素剥离、ARIA 正则表达式、URL 黑名单 | `content-security.ts` (编译的二进制文件) |
| L4 | TestSavantAI ONNX 分类器 | `security-classifier.ts` (侧边栏代理,未编译) |
| L4b | Claude Haiku 转录分类器 | `security-classifier.ts` (侧边栏代理) |
| L5 | 金丝雀令牌泄漏检测 | `security.ts` |

L1-L3 检查在 **保存时** 运行(在守护进程中)。L4 ML 分类器在 **加载时** 运行(在侧边栏代理中),因此每个将技能加载到其提示中的会话也会重新验证内容。这可以捕获仅在分类器模型更新后才显现的问题。

保存命令从 **活动标签页的顶级来源** 派生主机名,而不是从代理参数派生。这解决了 Codex 标记的混淆代理错误:恶意页面重定向链否则可能会欺骗代理污染不同的域。

## 错误参考

| 错误 | 原因 | 操作 |
|-------|-------|--------|
| `Save blocked: classifier flagged content as potential injection` | 保存时 L4 分数 ≥ 0.85 | 重写技能,删除类似指令的文本;重试。 |
| `Save blocked: <L1-L3 message>` | 保存时 URL 黑名单匹配或 ARIA 注入 | 检查技能正文中的可疑模式。 |
| `Save failed: empty body` | 通过 stdin 或 `--from-file` 没有内容 | 将 markdown 通过管道传输到 `$B domain-skill save`,或传递 `--from-file <path>`。 |
| `Cannot save domain-skill: no top-level URL on active tab` | 标签页是 `about:blank` 或 `chrome://...` | 先执行 `$B goto <target-site>`,然后保存。 |
| `Cannot promote: skill is in state "quarantined"` | 技能尚未自动提升 | 在此项目中使用它,直到 3 次成功运行且无分类器标记。 |
| `Cannot rollback: <host> has fewer than 2 versions` | 只存在一个版本 | 使用 `$B domain-skill rm` 删除。 |

## 遥测

当遥测启用时(默认 `community` 模式,除非关闭),以下事件会写入 `~/.gstack/analytics/browse-telemetry.jsonl`:

- `domain_skill_saved {host, scope, state, bytes}`
- `domain_skill_save_blocked {host, reason}`
- `domain_skill_fired {host, source, version}`
- `domain_skill_state_changed {host, from_state, to_state}` (计划中)

仅主机名 — 无正文内容,无代理文本。使用 `gstack-config set telemetry off` 或 `GSTACK_TELEMETRY_OFF=1` 完全禁用。