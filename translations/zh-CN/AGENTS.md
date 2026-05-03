# gstack — AI 工程工作流

gstack 是一组 SKILL.md 文件的集合,为 AI 代理提供结构化的软件开发角色。每个技能都是一个专家:CEO 审查员、工程经理、设计师、QA 负责人、发布工程师、调试器等等。

## 可用技能

技能文件位于 `.agents/skills/`(或 Claude Code 上的 `~/.claude/skills/gstack/`)。
通过名称调用它们(例如 `/office-hours`)。

### 计划模式审查

| 技能 | 功能说明 |
|-------|-------------|
| `/office-hours` | 从这里开始。在编写代码之前重新构思你的产品想法。 |
| `/plan-ceo-review` | CEO 级别审查:在需求中找到 10 星级产品。 |
| `/plan-eng-review` | 锁定架构、数据流、边界情况和测试。 |
| `/plan-design-review` | 对每个设计维度评分 0-10,解释 10 分是什么样子。 |
| `/plan-devex-review` | DX 模式审查:TTHW、神奇时刻、摩擦点、角色追踪。 |
| `/plan-tune` | 针对每个问题自调整 AskUserQuestion 敏感度。 |
| `/autoplan` | 一条命令运行 CEO → 设计 → 工程 → DX 审查。 |
| `/design-consultation` | 从零开始构建完整的设计系统。 |

### 实现 + 审查

| 技能 | 功能说明 |
|-------|-------------|
| `/review` | 合并前 PR 审查。发现通过 CI 但在生产环境中会出错的 bug。 |
| `/codex` | 通过 OpenAI Codex 获取第二意见。审查、质疑或咨询模式。 |
| `/investigate` | 系统化根因调试。没有调查就不修复。 |
| `/design-review` | 线上站点视觉审计 + 原子提交修复循环。 |
| `/design-shotgun` | 生成多个 AI 设计变体、对比面板、迭代。 |
| `/design-html` | 生成生产级 Pretext 原生 HTML/CSS。 |
| `/devex-review` | 实时开发者体验审计(根据实际流程测量 TTHW)。 |
| `/qa` | 打开真实浏览器,发现 bug,修复它们,重新验证。 |
| `/qa-only` | 与 /qa 相同的方法论,但仅报告 — 不更改代码。 |
| `/scrape` | 从网页提取数据。首次调用原型化;编码化调用在约 200ms 内运行。 |
| `/skillify` | 将最近成功的 `/scrape` 流程编码为永久的浏览器技能。 |

### 发布 + 部署

| 技能 | 功能说明 |
|-------|-------------|
| `/ship` | 运行测试、审查、推送、打开 PR。工作区感知版本队列。 |
| `/land-and-deploy` | 合并 PR,等待 CI 和部署,验证生产环境健康状况。 |
| `/canary` | 使用浏览守护进程的部署后监控循环。 |
| `/landing-report` | 工作区感知发布队列的只读仪表板。 |
| `/document-release` | 更新所有文档以匹配你刚刚发布的内容。 |
| `/setup-deploy` | 一次性部署配置检测(Fly.io、Render、Vercel 等)。 |
| `/gstack-upgrade` | 将 gstack 更新到最新版本。 |

### 运维 + 记忆

| 技能 | 功能说明 |
|-------|-------------|
| `/context-save` | 保存工作上下文(git 状态、决策、剩余工作)。 |
| `/context-restore` | 从保存的上下文恢复,甚至跨 Conductor 工作区。 |
| `/learn` | 管理 gstack 跨会话学到的内容。 |
| `/retro` | 每周回顾,包含每人细分和发布连续记录。 |
| `/health` | 代码质量仪表板(类型检查器、linter、测试、死代码)。 |
| `/benchmark` | 性能回归检测(页面加载、Core Web Vitals)。 |
| `/benchmark-models` | 技能的跨模型基准测试(Claude、GPT、Gemini 并排比较)。 |
| `/cso` | OWASP Top 10 + STRIDE 安全审计。 |
| `/setup-gbrain` | 设置 gbrain 用于跨机器会话内存同步。 |

### 浏览器 + 代理集成

| 技能 | 功能说明 |
|-------|-------------|
| `/browse` | 无头浏览器 — 真实 Chromium,真实点击,约 100ms/命令。 |
| `/open-gstack-browser` | 启动带侧边栏 + 隐身模式的可见 GStack 浏览器。 |
| `/setup-browser-cookies` | 从你的真实浏览器导入 cookies 用于认证测试。 |
| `/pair-agent` | 将远程 AI 代理(OpenClaw、Codex 等)与你的浏览器配对。 |

### 安全 + 范围限定

| 技能 | 功能说明 |
|-------|-------------|
| `/careful` | 在破坏性命令前警告(rm -rf、DROP TABLE、force-push)。 |
| `/freeze` | 锁定编辑到一个目录。硬阻止,不仅仅是警告。 |
| `/guard` | 同时激活 careful + freeze。 |
| `/unfreeze` | 移除目录编辑限制。 |
| `/make-pdf` | 将任何 markdown 文件转换为出版级 PDF。 |

## 构建命令

```bash
bun install              # install dependencies
bun test                 # run free tests (no API spend)
bun run test:windows     # curated Windows-safe subset (runs on windows-latest)
bun run build            # generate docs + compile binaries
bun run gen:skill-docs   # regenerate SKILL.md files from templates
bun run skill:check      # health dashboard for all skills
```

## 平台支持

- **macOS** + **Linux**: 支持完整测试套件。
- **Windows**: 精选的 Windows 安全子集在 `windows-latest` 上通过 `windows-free-tests` CI 作业运行。设置脚本(`./setup`)目前需要 Git Bash 或 MSYS;原生 PowerShell 支持是未来的扩展。`bin/gstack-paths` 辅助工具通过 `CLAUDE_PLUGIN_DATA` / `GSTACK_HOME` 解析状态根目录,因此插件安装在每个平台上都能工作。

## 关键约定

- SKILL.md 文件是从 `.tmpl` 模板**生成**的。编辑模板,而不是输出。
- 运行 `bun run gen:skill-docs --host codex` 重新生成 Codex 特定的输出。
- browse 二进制文件提供无头浏览器访问。在技能中使用 `$B <command>`。
- 安全技能(careful、freeze、guard)使用内联建议性文本 — 在破坏性操作前始终确认。
- 状态路径通过 `bin/gstack-paths` 解析(通过 `eval "$(...))"` 引入)。遵循 `GSTACK_HOME`、`CLAUDE_PLUGIN_DATA`、`CLAUDE_PLANS_DIR`。
- `claude` CLI 二进制文件通过 `browse/src/claude-bin.ts`(`Bun.which()` + `GSTACK_CLAUDE_BIN` 覆盖)解析。在 Windows 上设置 `GSTACK_CLAUDE_BIN=wsl` 加上 `GSTACK_CLAUDE_BIN_ARGS='["claude"]'` 以通过 WSL 运行 Claude。