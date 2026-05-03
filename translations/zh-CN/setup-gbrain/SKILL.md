```markdown
---
name: setup-gbrain
preamble-tier: 2
version: 1.0.0
description: |
  为此编码代理设置 gbrain：安装 CLI、初始化本地 PGLite 或 Supabase brain、
  注册 MCP、捕获每个远程的信任策略。一条命令从零到"gbrain 正在运行,此代理可以调用它"。
  使用场景："setup gbrain"、"connect gbrain"、"start gbrain"、
  "install gbrain"、"configure gbrain for this machine"。(gstack)
triggers:
  - setup gbrain
  - install gbrain
  - connect gbrain
  - start gbrain
  - configure gbrain
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
---
<!-- 从 SKILL.md.tmpl 自动生成 — 请勿直接编辑 -->
<!-- 重新生成: bun run gen:skill-docs -->

## Preamble (首先运行)

```bash
_UPD=$(~/.claude/skills/gstack/bin/gstack-update-check 2>/dev/null || .claude/skills/gstack/bin/gstack-update-check 2>/dev/null || true)
[ -n "$_UPD" ] && echo "$_UPD" || true
mkdir -p ~/.gstack/sessions
touch ~/.gstack/sessions/"$PPID"
_SESSIONS=$(find ~/.gstack/sessions -mmin -120 -type f 2>/dev/null | wc -l | tr -d ' ')
find ~/.gstack/sessions -mmin +120 -type f -exec rm {} + 2>/dev/null || true
_PROACTIVE=$(~/.claude/skills/gstack/bin/gstack-config get proactive 2>/dev/null || echo "true")
_PROACTIVE_PROMPTED=$([ -f ~/.gstack/.proactive-prompted ] && echo "yes" || echo "no")
_BRANCH=$(git branch --show-current 2>/dev/null || echo "unknown")
echo "BRANCH: $_BRANCH"
_SKILL_PREFIX=$(~/.claude/skills/gstack/bin/gstack-config get skill_prefix 2>/dev/null || echo "false")
echo "PROACTIVE: $_PROACTIVE"
echo "PROACTIVE_PROMPTED: $_PROACTIVE_PROMPTED"
echo "SKILL_PREFIX: $_SKILL_PREFIX"
source <(~/.claude/skills/gstack/bin/gstack-repo-mode 2>/dev/null) || true
REPO_MODE=${REPO_MODE:-unknown}
echo "REPO_MODE: $REPO_MODE"
_LAKE_SEEN=$([ -f ~/.gstack/.completeness-intro-seen ] && echo "yes" || echo "no")
echo "LAKE_INTRO: $_LAKE_SEEN"
_TEL=$(~/.claude/skills/gstack/bin/gstack-config get telemetry 2>/dev/null || true)
_TEL_PROMPTED=$([ -f ~/.gstack/.telemetry-prompted ] && echo "yes" || echo "no")
_TEL_START=$(date +%s)
_SESSION_ID="$$-$(date +%s)"
echo "TELEMETRY: ${_TEL:-off}"
echo "TEL_PROMPTED: $_TEL_PROMPTED"
_EXPLAIN_LEVEL=$(~/.claude/skills/gstack/bin/gstack-config get explain_level 2>/dev/null || echo "default")
if [ "$_EXPLAIN_LEVEL" != "default" ] && [ "$_EXPLAIN_LEVEL" != "terse" ]; then _EXPLAIN_LEVEL="default"; fi
echo "EXPLAIN_LEVEL: $_EXPLAIN_LEVEL"
_QUESTION_TUNING=$(~/.claude/skills/gstack/bin/gstack-config get question_tuning 2>/dev/null || echo "false")
echo "QUESTION_TUNING: $_QUESTION_TUNING"
mkdir -p ~/.gstack/analytics
if [ "$_TEL" != "off" ]; then
echo '{"skill":"setup-gbrain","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
fi
for _PF in $(find ~/.gstack/analytics -maxdepth 1 -name '.pending-*' 2>/dev/null); do
  if [ -f "$_PF" ]; then
    if [ "$_TEL" != "off" ] && [ -x "~/.claude/skills/gstack/bin/gstack-telemetry-log" ]; then
      ~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type skill_run --skill _pending_finalize --outcome unknown --session-id "$_SESSION_ID" 2>/dev/null || true
    fi
    rm -f "$_PF" 2>/dev/null || true
  fi
  break
done
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" 2>/dev/null || true
_LEARN_FILE="${GSTACK_HOME:-$HOME/.gstack}/projects/${SLUG:-unknown}/learnings.jsonl"
if [ -f "$_LEARN_FILE" ]; then
  _LEARN_COUNT=$(wc -l < "$_LEARN_FILE" 2>/dev/null | tr -d ' ')
  echo "LEARNINGS: $_LEARN_COUNT entries loaded"
  if [ "$_LEARN_COUNT" -gt 5 ] 2>/dev/null; then
    ~/.claude/skills/gstack/bin/gstack-learnings-search --limit 3 2>/dev/null || true
  fi
else
  echo "LEARNINGS: 0"
fi
~/.claude/skills/gstack/bin/gstack-timeline-log '{"skill":"setup-gbrain","event":"started","branch":"'"$_BRANCH"'","session":"'"$_SESSION_ID"'"}' 2>/dev/null &
_HAS_ROUTING="no"
if [ -f CLAUDE.md ] && grep -q "## Skill routing" CLAUDE.md 2>/dev/null; then
  _HAS_ROUTING="yes"
fi
_ROUTING_DECLINED=$(~/.claude/skills/gstack/bin/gstack-config get routing_declined 2>/dev/null || echo "false")
echo "HAS_ROUTING: $_HAS_ROUTING"
echo "ROUTING_DECLINED: $_ROUTING_DECLINED"
_VENDORED="no"
if [ -d ".claude/skills/gstack" ] && [ ! -L ".claude/skills/gstack" ]; then
  if [ -f ".claude/skills/gstack/VERSION" ] || [ -d ".claude/skills/gstack/.git" ]; then
    _VENDORED="yes"
  fi
fi
echo "VENDORED_GSTACK: $_VENDORED"
echo "MODEL_OVERLAY: claude"
_CHECKPOINT_MODE=$(~/.claude/skills/gstack/bin/gstack-config get checkpoint_mode 2>/dev/null || echo "explicit")
_CHECKPOINT_PUSH=$(~/.claude/skills/gstack/bin/gstack-config get checkpoint_push 2>/dev/null || echo "false")
echo "CHECKPOINT_MODE: $_CHECKPOINT_MODE"
echo "CHECKPOINT_PUSH: $_CHECKPOINT_PUSH"
[ -n "$OPENCLAW_SESSION" ] && echo "SPAWNED_SESSION: true" || true
```

## Plan Mode Safe Operations

在计划模式下,允许以下操作因为它们为计划提供信息:`$B`、`$D`、`codex exec`/`codex review`、写入 `~/.gstack/`、写入计划文件,以及对生成的产物使用 `open`。

## Skill Invocation During Plan Mode

如果用户在计划模式下调用技能,技能优先于通用计划模式行为。**将技能文件视为可执行指令,而非参考资料。**从步骤 0 开始逐步执行;第一个 AskUserQuestion 是工作流进入计划模式,而非违反计划模式。AskUserQuestion(任何变体 — `mcp__*__AskUserQuestion` 或原生;参见"AskUserQuestion Format → Tool resolution")满足计划模式的回合结束要求。如果没有可调用的变体,回退到将决策简报写入计划文件作为 `## Decisions to confirm` 部分 + ExitPlanMode — 永远不要静默自动决策。在 STOP 点,立即停止。不要继续工作流或在那里调用 ExitPlanMode。标记为"PLAN MODE EXCEPTION — ALWAYS RUN"的命令会执行。仅在技能工作流完成后,或用户告诉你取消技能或离开计划模式时,才调用 ExitPlanMode。

如果 `PROACTIVE` 是 `"false"`,不要自动调用或主动建议技能。如果某个技能看起来有用,询问:"我认为 /skillname 可能在这里有帮助 — 要我运行它吗?"

如果 `SKILL_PREFIX` 是 `"true"`,建议/调用 `/gstack-*` 名称。磁盘路径保持 `~/.claude/skills/gstack/[skill-name]/SKILL.md`。

如果输出显示 `UPGRADE_AVAILABLE <old> <new>`:读取 `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` 并遵循"Inline upgrade flow"(如果配置了自动升级,否则使用 4 个选项的 AskUserQuestion,如果拒绝则写入延迟状态)。

如果输出显示 `JUST_UPGRADED <from> <to>`:打印"Running gstack v{to} (just updated!)"。如果 `SPAWNED_SESSION` 为 true,跳过功能发现。

功能发现,每个会话最多一次提示:
- 缺少 `~/.claude/skills/gstack/.feature-prompted-continuous-checkpoint`:为 Continuous checkpoint 自动提交使用 AskUserQuestion。如果接受,运行 `~/.claude/skills/gstack/bin/gstack-config set checkpoint_mode continuous`。始终创建标记。
- 缺少 `~/.claude/skills/gstack/.feature-prompted-model-overlay`:通知"Model overlays are active. MODEL_OVERLAY shows the patch."始终创建标记。

升级提示后,继续工作流。

如果 `WRITING_STYLE_PENDING` 是 `yes`:询问一次关于写作风格:

> v1 提示更简单:首次使用术语注释、以结果为导向的问题、更短的文本。保持默认还是恢复简洁模式?

选项:
- A) 保持新的默认设置(推荐 — 好的写作对每个人都有帮助)
- B) 恢复 V0 文本 — 设置 `explain_level: terse`

如果 A:保持 `explain_level` 未设置(默认为 `default`)。
如果 B:运行 `~/.claude/skills/gstack/bin/gstack-config set explain_level terse`。

始终运行(无论选择如何):
```bash
rm -f ~/.gstack/.writing-style-prompt-pending
touch ~/.gstack/.writing-style-prompted
```

如果 `WRITING_STYLE_PENDING` 是 `no` 则跳过。

如果 `LAKE_INTRO` 是 `no`:说"gstack 遵循 **Boil the Lake** 原则 — 当 AI 使边际成本接近零时,做完整的事情。阅读更多:https://garryslist.org/posts/boil-the-ocean" 提供打开:

```bash
open https://garryslist.org/posts/boil-the-ocean
touch ~/.gstack/.completeness-intro-seen
```

仅在同意时运行 `open`。始终运行 `touch`。

如果 `TEL_PROMPTED` 是 `no` 且 `LAKE_INTRO` 是 `yes`:通过 AskUserQuestion 询问一次遥测:

> 帮助 gstack 变得更好。仅分享使用数据:技能、持续时间、崩溃、稳定设备 ID。不包括代码、文件路径或仓库名称。

选项:
- A) 帮助 gstack 变得更好!(推荐)
- B) 不用了,谢谢

如果 A:运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry community`

如果 B:询问后续:

> 匿名模式仅发送聚合使用情况,无唯一 ID。

选项:
- A) 好的,匿名可以
- B) 不用了,谢谢,完全关闭

如果 B→A:运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry anonymous`
如果 B→B:运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry off`

始终运行:
```bash
touch ~/.gstack/.telemetry-prompted
```

如果 `TEL_PROMPTED` 是 `yes` 则跳过。

如果 `PROACTIVE_PROMPTED` 是 `no` 且 `TEL_PROMPTED` 是 `yes`:询问一次:

> 让 gstack 主动建议技能,比如对"这能工作吗?"使用 /qa 或对 bug 使用 /investigate?

选项:
- A) 保持开启(推荐)
- B) 关闭它 — 我会自己输入 /commands

如果 A:运行 `~/.claude/skills/gstack/bin/gstack-config set proactive true`
如果 B:运行 `~/.claude/skills/gstack/bin/gstack-config set proactive false`

始终运行:
```bash
touch ~/.gstack/.proactive-prompted
```

如果 `PROACTIVE_PROMPTED` 是 `yes` 则跳过。

如果 `HAS_ROUTING` 是 `no` 且 `ROUTING_DECLINED` 是 `false` 且 `PROACTIVE_PROMPTED` 是 `yes`:
检查项目根目录中是否存在 CLAUDE.md 文件。如果不存在,创建它。

使用 AskUserQuestion:

> 当你的项目的 CLAUDE.md 包含技能路由规则时,gstack 效果最好。

选项:
- A) 向 CLAUDE.md 添加路由规则(推荐)
- B) 不用了,谢谢,我会手动调用技能

如果 A:将此部分附加到 CLAUDE.md 末尾:

```markdown

## Skill routing

当用户的请求匹配可用技能时,通过 Skill 工具调用它。如有疑问,调用技能。

关键路由规则:
- 产品想法/头脑风暴 → 调用 /office-hours
- 策略/范围 → 调用 /plan-ceo-review
- 架构 → 调用 /plan-eng-review
- 设计系统/计划审查 → 调用 /design-consultation 或 /plan-design-review
- 完整审查流程 → 调用 /autoplan
- Bug/错误 → 调用 /investigate
- QA/测试站点行为 → 调用 /qa 或 /qa-only
- 代码审查/差异检查 → 调用 /review
- 视觉优化 → 调用 /design-review
- 发布/部署/PR → 调用 /ship 或 /land-and-deploy
- 保存进度 → 调用 /context-save
- 恢复上下文 → 调用 /context-restore
```

然后提交更改:`git add CLAUDE.md && git commit -m "chore: add gstack skill routing rules to CLAUDE.md"`

如果 B:运行 `~/.claude/skills/gstack/bin/gstack-config set routing_declined true` 并告诉他们可以使用 `gstack-config set routing_declined false` 重新启用。

这仅在每个项目发生一次。如果 `HAS_ROUTING` 是 `yes` 或 `ROUTING_DECLINED` 是 `true` 则跳过。

如果 `VENDORED_GSTACK` 是 `yes`,除非 `~/.gstack/.vendoring-warned-$SLUG` 存在,否则通过 AskUserQuestion 警告一次:

> 此项目在 `.claude/skills/gstack/` 中有 gstack 供应商版本。供应商模式已弃用。
> 迁移到团队模式?

选项:
- A) 是的,现在迁移到团队模式
- B) 不,我会自己处理

如果 A:
1. 运行 `git rm -r .claude/skills/gstack/`
2. 运行 `echo '.claude/skills/gstack/' >> .gitignore`
3. 运行 `~/.claude/skills/gstack/bin/gstack-team-init required`(或 `optional`)
4. 运行 `git add .claude/ .gitignore CLAUDE.md && git commit -m "chore: migrate gstack from vendored to team mode"`
5. 告诉用户:"完成。每个开发者现在运行:`cd ~/.claude/skills/gstack && ./setup --team`"

如果 B:说"好的,你需要自己保持供应商副本的更新。"

始终运行(无论选择如何):
```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" 2>/dev/null || true
touch ~/.gstack/.vendoring-warned-${SLUG:-unknown}
```

如果标记存在,跳过。

如果 `SPAWNED_SESSION` 是 `"true"`,你正在由 AI 编排器(例如 OpenClaw)生成的会话中运行。在生成的会话中:
- 不要使用 AskUserQuestion 进行交互式提示。自动选择推荐选项。
- 不要运行升级检查、遥测提示、路由注入或 lake 介绍。
- 专注于完成任务并通过文本输出报告结果。
- 以完成报告结束:发布了什么、做出了什么决策、有什么不确定的。

## AskUserQuestion Format

### Tool resolution (首先阅读)

"AskUserQuestion" 在运行时可以解析为两个工具:**宿主 MCP 变体**(例如 `mcp__conductor__AskUserQuestion` — 当宿主注册它时出现在你的工具列表中)或 **原生** Claude Code 工具。

**规则:**如果你的工具列表中有任何 `mcp__*__AskUserQuestion` 变体,优先使用它。宿主可能通过 `--disallowedTools AskUserQuestion` 禁用原生 AUQ(Conductor 默认这样做)并通过其 MCP 变体路由;在那里调用原生会静默失败。相同的问题/选项形状;相同的决策简报格式适用。

**当两个变体都不可调用时的回退:**在计划模式下,将决策简报写入计划文件作为 `## Decisions to confirm` 部分 + ExitPlanMode(原生"Ready to execute?"会显示它)。在计划模式之外,将简报作为文本输出并停止。**永远不要静默自动决策** — 只有 `/plan-tune` AUTO_DECIDE 选择授权自动选择。

### Format

每个 AskUserQuestion 都是一个决策简报,必须作为 tool_use 发送,而非文本。

```
D<N> — <单行问题标题>
Project/branch/task: <使用 _BRANCH 的 1 个简短定位句子>
ELI10: <16 岁的人能理解的简单英语,2-4 句话,说明利害关系>
Stakes if we pick wrong: <关于什么会损坏、用户会看到什么、会失去什么的一句话>
Recommendation: <choice> because <单行原因>
Completeness: A=X/10, B=Y/10   (或: Note: options differ in kind, not coverage — no completeness score)
Pros / cons:
A) <选项标签> (recommended)
  ✅ <优点 — 具体、可观察、≥40 字符>
  ❌ <缺点 — 诚实、≥40 字符>
B) <选项标签>
  ✅ <优点>
  ❌ <缺点>
Net: <对你实际权衡内容的单行综合>
```

D 编号:技能调用中的第一个问题是 `D1`;自己递增。这是模型级指令,而非运行时计数器。

ELI10 始终存在,用简单英语,而非函数名。Recommendation 始终存在。保留 `(recommended)` 标签;AUTO_DECIDE 依赖它。

Completeness:仅当选项在覆盖范围上不同时使用 `Completeness: N/10`。10 = 完整,7 = 快乐路径,3 = 捷径。如果选项在类型上不同,写:`Note: options differ in kind, not coverage — no completeness score.`

Pros / cons:使用 ✅ 和 ❌。当选择是真实的时,每个选项至少 2 个优点和 1 个缺点;每个要点至少 40 个字符。单向/破坏性确认的硬停止转义:`✅ No cons — this is a hard-stop choice`。

中立姿态:`Recommendation: <default> — this is a taste call, no strong preference either way`;`(recommended)` 保留在默认选项上以供 AUTO_DECIDE 使用。

Effort both-scales:当选项涉及工作量时,标记人类团队和 CC+gstack 时间,例如 `(human: ~2 days / CC: ~15 min)`。在决策时使 AI 压缩可见。

Net 行结束权衡。每个技能指令可能添加更严格的规则。

### Self-check before emitting

在调用 AskUserQuestion 之前,验证:
- [ ] D<N> 标题存在
- [ ] ELI10 段落存在(利害关系行也存在)
- [ ] Recommendation 行存在并有具体原因
- [ ] Completeness 已评分(覆盖范围)或存在类型注释(类型)
- [ ] 每个选项都有 ≥2 个 ✅ 和 ≥1 个 ❌,每个 ≥40 字符(或硬停止转义)
- [ ] 一个选项上有 (recommended) 标签(即使是中立姿态)
- [ ] 涉及工作量的选项上有双尺度工作量标签(human / CC)
- [ ] Net 行结束决策
- [ ] 你正在调用工具,而非写文本


## GBrain Sync (技能开始)

```bash
_GSTACK_HOME="${GSTACK_HOME:-$HOME/.gstack}"
_BRAIN_REMOTE_FILE="$HOME/.gstack-brain-remote.txt"
_BRAIN_SYNC_BIN="~/.claude/skills/gstack/bin/gstack-brain-sync"
_BRAIN_CONFIG_BIN="~/.claude/skills/gstack/bin/gstack-config"

_BRAIN_SYNC_MODE=$("$_BRAIN_CONFIG_BIN" get gbrain_sync_mode 2>/dev/null || echo off)

if [ -f "$_BRAIN_REMOTE_FILE" ] && [ ! -d "$_GSTACK_HOME/.git" ] && [ "$_BRAIN_SYNC_MODE" = "off" ]; then
  _BRAIN_NEW_URL=$(head -1 "$_BRAIN_REMOTE_FILE" 2>/dev/null | tr -d '[:space:]')
  if [ -n "$_BRAIN_NEW_URL" ]; then
    echo "BRAIN_SYNC: brain repo detected: $_BRAIN_NEW_URL"
    echo "BRAIN_SYNC: run 'gstack-brain-restore' to pull your cross-machine memory (or 'gstack-config set gbrain_sync_mode off' to dismiss forever)"
  fi
fi

if [ -d "$_GSTACK_HOME/.git" ] && [ "$_BRAIN_SYNC_MODE" != "off" ]; then
  _BRAIN_LAST_PULL_FILE="$_GSTACK_HOME/.brain-last-pull"
  _BRAIN_NOW=$(date +%s)
  _BRAIN_DO_PULL=1
  if [ -f "$_BRAIN_LAST_PULL_FILE" ]; then
    _BRAIN_LAST=$(cat "$_BRAIN_LAST_PULL_FILE" 2>/dev/null || echo 0)
    _BRAIN_AGE=$(( _BRAIN_NOW - _BRAIN_LAST ))
    [ "$_BRAIN_AGE" -lt 86400 ] && _BRAIN_DO_PULL=0
  fi
  if [ "$_BRAIN_DO_PULL" = "1" ]; then
    ( cd "$_GSTACK_HOME" && git fetch origin >/dev/null 2>&1 && git merge --ff-only "origin/$(git rev-parse --abbrev-ref HEAD)" >/dev/null 2>&1 ) || true
    echo "$_BRAIN_NOW" > "$_BRAIN_LAST_PULL_FILE"
  fi
  "$_BRAIN_SYNC_BIN" --once 2>/dev/null || true
fi

if [ -d "$_GSTACK_HOME/.git" ] && [ "$_BRAIN_SYNC_MODE" != "off" ]; then
  _BRAIN_QUEUE_DEPTH=0
  [ -f "$_GSTACK_HOME/.brain-queue.jsonl" ] && _BRAIN_QUEUE_DEPTH=$(wc -l < "$_GSTACK_HOME/.brain-queue.jsonl" | tr -d ' ')
  _BRAIN_LAST_PUSH="never"
  [ -f "$_GSTACK_HOME/.brain-last-push" ] && _BRAIN_LAST_PUSH=$(cat "$_GSTACK_HOME/.brain-last-push" 2>/dev/null || echo never)
  echo "BRAIN_SYNC: mode=$_BRAIN_SYNC_MODE | last_push=$_BRAIN_LAST_PUSH | queue=$_BRAIN_QUEUE_DEPTH"
else
  echo "BRAIN_SYNC: off"
fi
```



隐私停止门:如果输出显示 `BRAIN_SYNC: off`,`gbrain_sync_mode_prompted` 是 `false`,且 gbrain 在 PATH 上或 `gbrain doctor --fast --json` 工作,询问一次:

> gstack 可以将你的会话记忆发布到 GBrain 跨机器索引的私有 GitHub 仓库。应该同步多少?

选项:
- A) 所有允许列表的内容(推荐)
- B) 仅产物
- C) 拒绝,保持所有内容本地

回答后:

```bash
# 选择的模式: full | artifacts-only | off
"$_BRAIN_CONFIG_BIN" set gbrain_sync_mode <choice>
"$_BRAIN_CONFIG_BIN" set gbrain_sync_mode_prompted true
```

如果 A/B 且 `~/.gstack/.git` 缺失,询问是否运行 `gstack-brain-init`。不要阻塞技能。

在遥测之前的技能结束时:

```bash
"~/.claude/skills/gstack/bin/gstack-brain-sync" --discover-new 2>/dev/null || true
"~/.claude/skills/gstack/bin/gstack-brain-sync" --once 2>/dev/null || true
```


## Model-Specific Behavioral Patch (claude)

以下提示针对 claude 模型系列进行了调整。它们**从属于**技能工作流、STOP 点、AskUserQuestion 门、计划模式安全和 /ship 审查门。如果下面的提示与技能指令冲突,技能获胜。将这些视为偏好,而非规则。

**待办事项列表纪律。**在执行多步骤计划时,在完成每个任务时单独标记为完成。不要在最后批量完成。如果任务被证明是不必要的,用单行原因标记为跳过。

**在重大操作之前思考。**对于复杂操作(重构、迁移、非平凡的新功能),在执行前简要说明你的方法。这让用户可以廉价地纠正方向,而不是在执行过程中。

**专用工具优于 Bash。**优先使用 Read、Edit、Write、Glob、Grep 而非 shell 等效命令(cat、sed、find、grep)。专用工具更便宜、更清晰。

## Voice

GStack 语音:Garry 形式的产品和工程判断,为运行时压缩。

- 以要点开头。说明它做什么、为什么重要以及对构建者有什么变化。
- 具体。命名文件、函数、行号、命令、输出、评估和实际数字。
- 将技术选择与用户结果联系起来:真实用户看到什么、失去什么、等待什么或现在可以做什么。
- 对质量直接。Bug 很重要。边缘情况很重要。修复整个事情,而非演示路径。
- 听起来像构建