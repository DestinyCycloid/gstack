# 设计:在 /review 和 /ship 中集成 slop-scan

状态:已推迟
创建时间:2026-04-09
依赖:slop-diff 脚本(scripts/slop-diff.ts,已合并)

## 问题

slop-scan 的发现结果只有在手动运行 `bun run slop:diff` 时才可见。它们应该在代码审查和发布过程中自动显示,就像 SQL 安全检查和信任边界检查一样。

## 集成点

### /review(第 4 步,检查清单通过后)

在关键/信息性检查清单通过后运行 `bun run slop:diff`。将新发现的问题与其他审查输出一起内联显示:

```
Pre-Landing Review: 3 issues (1 critical, 2 informational)

AI Slop: +2 new findings, -0 removed
  browse/src/new-feature.ts
    defensive.empty-catch: 2 locations
      line 42: empty catch, boundary=filesystem
      line 87: empty catch, boundary=process
```

分类:INFORMATIONAL(永远不会阻止合并,只是显示模式)。

应用 Fix-First 启发式规则:如果发现的问题是文件操作周围的空 catch,则使用 `safeUnlink()` 自动修复。如果是扩展代码中的 catch-and-log,则跳过(根据 CLAUDE.md 指南,这是正确的模式)。

### /ship(第 3.5 步,合并前审查 + PR 正文)

与 /review 相同的集成。此外,在 PR 正文中显示一行摘要:

```markdown
## Pre-Landing Review
- 2 issues auto-fixed, 0 needs input
- AI Slop: +0 new / -3 removed ✓
```

### Review Readiness Dashboard

不要添加行。Slop 是对差异的诊断,而不是独立"运行"的审查。它显示在 Eng Review 输出内部,而不是作为自己的仪表板条目。

## 什么需要自动修复 vs 什么需要跳过

遵循 CLAUDE.md 的"Slop-scan"部分。摘要:

**自动修复(真正的质量改进):**
- `fs.unlinkSync` 周围的空 catch → 替换为 `safeUnlink()`
- `process.kill` 周围的空 catch → 替换为 `safeKill()`
- 没有外层 try 的 `return await` → 移除 `await`
- URL 解析周围的无类型 catch → 添加 `instanceof TypeError` 检查

**跳过(slop-scan 标记的正确模式):**
- fire-and-forget 浏览器操作上的 `.catch(() => {})`(page.close、bringToFront)
- Chrome 扩展代码中的 catch-and-log(未捕获的错误会导致扩展崩溃)
- 关闭/紧急路径中的 `safeUnlinkQuiet`(吞掉所有错误是正确的)
- 委托给活动会话的传递包装器(API 稳定性层)

## 实现说明

- `scripts/slop-diff.ts` 已经处理了繁重的工作(基于工作树的基准比较、行号不敏感的指纹识别、优雅降级)
- review/ship 技能运行 bash 块。集成方式是:运行脚本,解析输出,包含在审查发现中
- 如果未安装 slop-scan(`npx slop-scan` 失败),则静默跳过
- 脚本始终以 0 退出(诊断性质,永不阻塞)

## 工作量估算

| 任务 | 人工 | CC+gstack |
|------|-------|-----------|
| 添加到 review/SKILL.md.tmpl | 2 小时 | 10 分钟 |
| 添加到 ship/SKILL.md.tmpl | 2 小时 | 10 分钟 |
| 添加到 review/checklist.md | 1 小时 | 5 分钟 |
| 使用实际 PR 测试 | 2 小时 | 15 分钟 |
| 重新生成 SKILL.md 文件 | — | 1 分钟 |