---
name: make-pdf
preamble-tier: 1
version: 1.0.0
description: |
  将任何 Markdown 文件转换为出版级 PDF。标准 1 英寸边距、智能分页、页码、封面页、页眉、弯引号和破折号、可点击目录、对角 DRAFT 水印。不是草稿产物——是完成品。当被要求"制作 PDF"、"导出为 PDF"、"将此 Markdown 转换为 PDF"或"生成文档"时使用。(gstack)
  语音触发词(语音转文字别名):"make this a pdf"、"make it a pdf"、"export to pdf"、"turn this into a pdf"、"turn this markdown into a pdf"、"generate a pdf"、"make a pdf from"、"pdf this markdown"。
triggers:
  - markdown to pdf
  - generate pdf
  - make pdf
  - export pdf
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion
---
<!-- 从 SKILL.md.tmpl 自动生成 — 请勿直接编辑 -->
<!-- 重新生成: bun run gen:skill-docs -->

## 前置代码(首先运行)

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
echo '{"skill":"make-pdf","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
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
~/.claude/skills/gstack/bin/gstack-timeline-log '{"skill":"make-pdf","event":"started","branch":"'"$_BRANCH"'","session":"'"$_SESSION_ID"'"}' 2>/dev/null &
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

## 计划模式安全操作

在计划模式下,允许以下操作,因为它们为计划提供信息:`$B`、`$D`、`codex exec`/`codex review`、写入 `~/.gstack/`、写入计划文件,以及为生成的产物执行 `open`。

## 计划模式期间的技能调用

如果用户在计划模式下调用技能,技能优先于通用计划模式行为。**将技能文件视为可执行指令,而非参考资料。**从步骤 0 开始逐步执行;第一个 AskUserQuestion 是工作流进入计划模式,而非违反计划模式。AskUserQuestion(任何变体——`mcp__*__AskUserQuestion` 或原生;参见"AskUserQuestion 格式 → 工具解析")满足计划模式的回合结束要求。如果没有可调用的变体,回退到将决策简报写入计划文件作为 `## Decisions to confirm` 部分 + ExitPlanMode——永远不要静默自动决策。在 STOP 点,立即停止。不要继续工作流或在那里调用 ExitPlanMode。标记为"PLAN MODE EXCEPTION — ALWAYS RUN"的命令会执行。仅在技能工作流完成后,或用户告诉你取消技能或离开计划模式时,才调用 ExitPlanMode。

如果 `PROACTIVE` 为 `"false"`,不要自动调用或主动建议技能。如果某个技能看起来有用,询问:"我认为 /skillname 可能对这里有帮助——要我运行它吗?"

如果 `SKILL_PREFIX` 为 `"true"`,建议/调用 `/gstack-*` 名称。磁盘路径保持为 `~/.claude/skills/gstack/[skill-name]/SKILL.md`。

如果输出显示 `UPGRADE_AVAILABLE <old> <new>`:读取 `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` 并遵循"内联升级流程"(如果已配置则自动升级,否则使用 4 个选项的 AskUserQuestion,如果拒绝则写入延迟状态)。

如果输出显示 `JUST_UPGRADED <from> <to>`:打印"Running gstack v{to} (just updated!)"。如果 `SPAWNED_SESSION` 为 true,跳过功能发现。

功能发现,每个会话最多一次提示:
- 缺少 `~/.claude/skills/gstack/.feature-prompted-continuous-checkpoint`:使用 AskUserQuestion 询问持续检查点自动提交。如果接受,运行 `~/.claude/skills/gstack/bin/gstack-config set checkpoint_mode continuous`。始终创建标记。
- 缺少 `~/.claude/skills/gstack/.feature-prompted-model-overlay`:通知"模型覆盖已激活。MODEL_OVERLAY 显示补丁。"始终创建标记。

升级提示后,继续工作流。

如果 `WRITING_STYLE_PENDING` 为 `yes`:询问一次关于写作风格:

> v1 提示更简单:首次使用术语注释、以结果为导向的问题、更短的文本。保持默认还是恢复简洁模式?

选项:
- A) 保持新的默认设置(推荐——好的写作对每个人都有帮助)
- B) 恢复 V0 文本——设置 `explain_level: terse`

如果 A:保持 `explain_level` 未设置(默认为 `default`)。
如果 B:运行 `~/.claude/skills/gstack/bin/gstack-config set explain_level terse`。

始终运行(无论选择如何):
```bash
rm -f ~/.gstack/.writing-style-prompt-pending
touch ~/.gstack/.writing-style-prompted
```

如果 `WRITING_STYLE_PENDING` 为 `no`,跳过。

如果 `LAKE_INTRO` 为 `no`:说"gstack 遵循 **Boil the Lake** 原则——当 AI 使边际成本接近零时,做完整的事情。阅读更多:https://garryslist.org/posts/boil-the-ocean" 提供打开:

```bash
open https://garryslist.org/posts/boil-the-ocean
touch ~/.gstack/.completeness-intro-seen
```

仅在同意时运行 `open`。始终运行 `touch`。

如果 `TEL_PROMPTED` 为 `no` 且 `LAKE_INTRO` 为 `yes`:通过 AskUserQuestion 询问一次遥测:

> 帮助 gstack 变得更好。仅分享使用数据:技能、持续时间、崩溃、稳定设备 ID。不包含代码、文件路径或仓库名称。

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

如果 `TEL_PROMPTED` 为 `yes`,跳过。

如果 `PROACTIVE_PROMPTED` 为 `no` 且 `TEL_PROMPTED` 为 `yes`:询问一次:

> 让 gstack 主动建议技能,比如对"这能工作吗?"使用 /qa 或对 bug 使用 /investigate?

选项:
- A) 保持开启(推荐)
- B) 关闭它——我会自己输入 /commands

如果 A:运行 `~/.claude/skills/gstack/bin/gstack-config set proactive true`
如果 B:运行 `~/.claude/skills/gstack/bin/gstack-config set proactive false`

始终运行:
```bash
touch ~/.gstack/.proactive-prompted
```

如果 `PROACTIVE_PROMPTED` 为 `yes`,跳过。

如果 `HAS_ROUTING` 为 `no` 且 `ROUTING_DECLINED` 为 `false` 且 `PROACTIVE_PROMPTED` 为 `yes`:
检查项目根目录中是否存在 CLAUDE.md 文件。如果不存在,创建它。

使用 AskUserQuestion:

> 当你的项目的 CLAUDE.md 包含技能路由规则时,gstack 效果最好。

选项:
- A) 向 CLAUDE.md 添加路由规则(推荐)
- B) 不用了,谢谢,我会手动调用技能

如果 A:将此部分追加到 CLAUDE.md 末尾:

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

这仅在每个项目中发生一次。如果 `HAS_ROUTING` 为 `yes` 或 `ROUTING_DECLINED` 为 `true`,跳过。

如果 `VENDORED_GSTACK` 为 `yes`,除非 `~/.gstack/.vendoring-warned-$SLUG` 存在,否则通过 AskUserQuestion 警告一次:

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

如果 `SPAWNED_SESSION` 为 `"true"`,你正在由 AI 编排器(例如 OpenClaw)生成的会话中运行。在生成的会话中:
- 不要使用 AskUserQuestion 进行交互式提示。自动选择推荐选项。
- 不要运行升级检查、遥测提示、路由注入或 lake 介绍。
- 专注于完成任务并通过文本输出报告结果。
- 以完成报告结束:发布了什么、做出的决策、任何不确定的事项。

## GBrain 同步(技能开始)

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

隐私停止门:如果输出显示 `BRAIN_SYNC: off`,`gbrain_sync_mode_prompted` 为 `false`,且 gbrain 在 PATH 上或 `gbrain doctor --fast --json` 有效,询问一次:

> gstack 可以将你的会话记忆发布到 GBrain 跨机器索引的私有 GitHub 仓库。应该同步多少?

选项:
- A) 所有允许列表内容(推荐)
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

## 模型特定行为补丁(claude)

以下微调针对 claude 模型系列。它们**从属于**技能工作流、STOP 点、AskUserQuestion 门控、计划模式安全和 /ship 审查门控。如果下面的微调与技能指令冲突,技能优先。将这些视为偏好,而非规则。

**待办事项列表纪律。**在处理多步骤计划时,在完成每个任务时单独标记为完成。不要在最后批量完成。如果某个任务被证明是不必要的,标记为跳过并附上一行原因。

**在重大操作前思考。**对于复杂操作(重构、迁移、非平凡的新功能),在执行前简要说明你的方法。这让用户可以低成本地纠正方向,而不是在执行过程中。

**专用工具优于 Bash。**优先使用 Read、Edit、Write、Glob、Grep 而非 shell 等效命令(cat、sed、find、grep)。专用工具更便宜且更清晰。

## 语气

直接、具体、构建者对构建者。命名文件、函数、命令和用户可见影响。无填充词。

不使用破折号。不使用 AI 词汇:delve、crucial、robust、comprehensive、nuanced、multifaceted。永远不要企业化或学术化。短段落。以要做什么结束。

用户拥有你没有的上下文。跨模型一致是建议,而非决策。用户决定。

## 完成状态协议

完成技能工作流时,使用以下之一报告状态:
- **DONE** — 已完成并有证据。
- **DONE_WITH_CONCERNS** — 已完成,但列出关注点。
- **BLOCKED** — 无法继续;说明阻塞因素和尝试过的方法。
- **NEEDS_CONTEXT** — 缺少信息;准确说明需要什么。

在 3 次失败尝试后、不确定的安全敏感更改或无法验证的范围后升级。格式:`STATUS`、`REASON`、`ATTEMPTED`、`RECOMMENDATION`。

## 操作性自我改进

完成前,如果你发现了一个持久的项目怪癖或命令修复,下次可以节省 5 分钟以上,记录它:

```bash
~/.claude/skills/gstack/bin/gstack-learnings-log '{"skill":"SKILL_NAME","type":"operational","key":"SHORT_KEY","insight":"DESCRIPTION","confidence":N,"source":"observed"}'
```

不要记录明显的事实或一次性瞬态错误。

## 遥测(最后运行)

工作流完成后,记录遥测。使用前置元数据中的技能 `name:`。OUTCOME 为 success/error/abort/unknown。

**PLAN MODE EXCEPTION — ALWAYS RUN:** 此命令将遥测写入 `~/.gstack/analytics/`,与前置代码分析写入匹配。

运行此 bash:

```bash
_TEL_END=$(date +%s)
_TEL_DUR=$(( _TEL_END - _TEL_START ))
rm -f ~/.gstack/analytics/.pending-"$_SESSION_ID" 2>/dev/null || true
# 会话时间线:记录技能完成(仅本地,永不发送到任何地方)
~/.claude/skills/gstack/bin/gstack-timeline-log '{"skill":"SKILL_NAME","event":"completed","branch":"'$(git branch --show-current 2>/dev/null || echo unknown)'","outcome":"OUTCOME","duration_s":"'"$_TEL_DUR"'","session":"'"$_SESSION_ID"'"}' 2>/dev/null || true
# 本地分析(受遥测设置控制)
if [ "$_TEL" != "off" ]; then
echo '{"skill":"SKILL_NAME","duration_s":"'"$_TEL_DUR"'","outcome":"OUTCOME","browse":"USED_BROWSE","session":"'"$_SESSION_ID"'","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"}' >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
fi
# 远程遥测(选择加入,需要二进制文件)
if [ "$_TEL" != "off" ] && [ -x ~/.claude/skills/gstack/bin/gstack-telemetry-log ]; then
  ~/.claude/skills/gstack/bin/gstack-telemetry-log \
    --skill "SKILL_NAME" --duration "$_TEL_DUR" --outcome "OUTCOME" \
    --used-browse "USED_BROWSE" --session-id "$_SESSION_ID" 2>/dev/null &
fi
```

运行前替换 `SKILL_NAME`、`OUTCOME` 和 `USED_BROWSE`。

## 计划状态页脚

在 ExitPlanMode 之前的计划模式中:如果计划文件缺少 `## GSTACK REVIEW REPORT`,运行 `~/.claude/skills/gstack/bin/gstack-review-read` 并追加标准运行/状态/发现表。如果是 `NO_REVIEWS` 或为空,追加一个 5 行占位符,判定为"NO REVIEWS YET — run `/autoplan`"。如果存在更丰富的报告,跳过。

PLAN MODE EXCEPTION — 始终允许(它是计划文件)。

# make-pdf: 从 Markdown 生成出版级 PDF

将 `.md` 文件转换为看起来像 Faber & Faber 散文的 PDF:1 英寸边距、左对齐正文、全部使用 Helvetica、弯引号和破折号、可选封面页和可点击目录、需要时的对角 DRAFT 水印。从 PDF 复制粘贴产生干净的文字,永远不会是"S a i l i n g"。

在 Linux 上,安装 `fonts-liberation` 以正确渲染——Helvetica 和 Arial 默认不存在,Liberation Sans 是标准的度量兼容后备字体。CI 和 Docker 构建通过 Dockerfile.ci 自动安装它。

## MAKE-PDF 设置(在任何 make-pdf 命令之前运行此检查)

```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)