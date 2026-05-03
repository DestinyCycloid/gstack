# 会话智能层

## 问题

Claude Code 的上下文窗口是临时的。每个会话都从头开始。当自动压缩在约 167K tokens 时触发,它会保留一个通用摘要,但会销毁文件读取、推理链和中间决策。

gstack 已经产生了在磁盘上持久化的有价值的产物:CEO 计划、工程评审、设计评审、QA 报告、学习记录。这些文件包含了塑造当前工作的决策、约束和上下文。但 Claude 不知道它们的存在。压缩后,那些指导每个决策的计划和评审会悄无声息地从上下文中消失。

生态系统正在解决这个问题。claude-mem(9K+ stars)捕获工具使用情况并将上下文注入到未来的会话中。Claude HUD 显示实时代理状态。Anthropic 自己的 `claude-progress.txt` 模式使用一个进度文件,代理在每个会话开始时读取它。

没有人在解决让**技能产生的产物**在压缩后存活的具体问题。因为没有其他人拥有 gstack 的产物架构。

## 洞察

gstack 已经将结构化产物写入 `~/.gstack/projects/$SLUG/`:
- CEO 计划:`ceo-plans/`
- 设计评审:`design-reviews/`
- 工程评审:`eng-reviews/`
- 学习记录:`learnings.jsonl`
- 技能使用:`../analytics/skill-usage.jsonl`

缺失的部分不是存储。而是感知。前言需要告诉代理:"这些文件存在。它们包含你已经做出的决策。压缩后,重新读取它们。"

## 架构

```
                   ┌─────────────────────────────────────┐
                   │        Claude 上下文窗口              │
                   │   (临时的,约 167K token 限制)         │
                   │                                      │
                   │   压缩触发 ──► 仅保留摘要              │
                   └──────────────┬──────────────────────┘
                                  │
                          启动时/压缩后读取
                                  │
                   ┌──────────────▼──────────────────────┐
                   │    ~/.gstack/projects/$SLUG/         │
                   │    (持久化,在任何情况下都存活)          │
                   │                                      │
                   │  ceo-plans/         ← /plan-ceo-review
                   │  eng-reviews/       ← /plan-eng-review
                   │  design-reviews/    ← /plan-design-review
                   │  checkpoints/       ← /checkpoint (新)
                   │  timeline.jsonl     ← 每个技能 (新)
                   │  learnings.jsonl    ← /learn
                   └─────────────────────────────────────┘
                                  │
                          每周汇总
                                  │
                   ┌──────────────▼──────────────────────┐
                   │           /retro                      │
                   │  时间线: 3 /review, 2 /ship, ...      │
                   │  健康趋势: compile 8/10 (↑2)          │
                   │  应用的学习: 本周 4 条                 │
                   └─────────────────────────────────────┘
```

## 功能

### 第 1 层:上下文恢复(前言,所有技能)
前言中约 10 行文字。压缩或上下文退化后,代理检查 `~/.gstack/projects/$SLUG/` 以查找最近的计划、评审和检查点。列出目录,读取最新文件。

成本:接近零。收益:每个技能的计划/评审在压缩后存活。

### 第 2 层:会话时间线(前言,所有技能)
每个技能向 `timeline.jsonl` 追加一行 JSONL 条目:时间戳、技能名称、分支、关键结果。`/retro` 渲染它。

使项目的 AI 辅助工作历史可见。"本周:3 /review、2 /ship、1 /investigate,跨分支 feature-auth 和 fix-billing。"

### 第 3 层:跨会话注入(前言,所有技能)
当新会话在具有最近产物的分支上启动时,前言打印一行:"上次会话:实现了 JWT 认证,完成 3/5 任务。计划:~/.gstack/projects/$SLUG/checkpoints/latest.md"

代理在读取任何文件之前就知道你上次停在哪里。

### 第 4 层:/checkpoint(可选技能)
工作状态的手动快照:正在做什么、正在编辑的文件、做出的决策、剩余工作。在离开之前、复杂操作之前、工作区交接或几天后回来时很有用。

### 第 5 层:/health(可选技能)
代码质量仪表板:类型检查、lint、测试套件、死代码扫描。综合 0-10 分。随时间跟踪。`/retro` 显示趋势。`/ship` 在可配置阈值上设置门控。

## 复合效应

每个功能都是独立有用的。它们一起创造了复合的东西:

会话 1:/plan-ceo-review 生成一个计划。保存到磁盘。
会话 2:代理在前言后读取计划。不会重新询问决策。
会话 3:/checkpoint 保存进度。时间线显示 2 /review、1 /ship。
会话 4:重构过程中触发压缩。代理重新读取检查点。
       恢复关键决策、类型、剩余工作。继续。
会话 5:/retro 汇总本周。健康趋势:6/10 → 8/10。
       时间线显示跨 3 个分支的 12 次技能调用。

项目的 AI 历史不再是临时的。它持久化、复合,并使每个未来会话更智能。这就是会话智能层。

## 这不是什么

- 不是 Claude 内置压缩的替代品(那处理会话状态;我们处理 gstack 产物)
- 不是像 claude-mem 那样的完整内存系统(那通过 SQLite 处理跨会话内存;我们处理结构化技能产物)
- 不是数据库或服务(只是磁盘上的 markdown 文件)

## 研究来源

- [Anthropic: Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Anthropic: Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [claude-mem](https://github.com/thedotmack/claude-mem)
- [Claude HUD](https://github.com/jarrodwatts/claude-hud)
- [CodeScene: Agentic AI coding best practices](https://codescene.com/blog/agentic-ai-coding-best-practice-patterns-for-speed-with-quality)
- [Post-compaction recovery via git-persisted state (Beads)](https://dev.to/jeremy_longshore/building-post-compaction-recovery-for-ai-agent-workflows-with-beads-207l)