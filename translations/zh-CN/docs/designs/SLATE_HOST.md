# Slate 宿主集成 — 研究与设计文档

**日期:** 2026-04-02
**分支:** garrytan/slate-agent-support
**状态:** 研究完成,受阻于宿主配置重构
**取代:** 无

## 什么是 Slate

Slate 是 Random Labs 的专有编码代理 CLI。
安装: `npm i -g @randomlabs/slate` 或 `brew install anthropic/tap/slate`。
许可证: 专有。85MB 编译的 Bun 二进制文件 (arm64/x64, darwin/linux/windows)。
npm 包: `@randomlabs/slate@1.0.25` (轻量 8.8KB 启动器 + 平台特定可选依赖)。

多模型: 动态选择 Claude Sonnet/Opus/Haiku,以及其他模型。
为"集群编排"构建,支持扩展的多小时会话。

## Slate 是 OpenCode 的分支

**通过 85MB Mach-O arm64 二进制文件的字符串分析确认:**

- 内部名称: `name: "opencode"` (二进制文件中的字面字符串)
- 所有 `OPENCODE_*` 环境变量与 `SLATE_*` 等效变量并存
- 共享 OpenCode 的工具/技能架构、LSP 集成、终端管理
- 拥有自己的品牌、API 端点 (`api.randomlabs.ai`, `agent-worker-prod.randomlabs.workers.dev`) 和配置路径

这对集成很重要: OpenCode 约定大多适用,但 Slate 在此基础上添加了自己的路径和环境变量。

## 技能发现(从二进制文件确认)

Slate 扫描所有四个目录系列以查找技能。二进制文件中的错误消息确认:

```
"failed .slate directory scan for skills"
"failed .claude directory scan for skills"
"failed .agents directory scan for skills"
"failed .opencode directory scan for skills"
```

**发现路径(来自 Slate 文档的优先级顺序):**

1. `.slate/skills/<name>/SKILL.md` — 项目级别,最高优先级
2. `~/.slate/skills/<name>/SKILL.md` — 全局
3. `.opencode/skills/`, `.agents/skills/` — 兼容性回退
4. `.claude/skills/` — Claude Code 兼容性回退(最低)
5. 通过 `slate.json` 自定义路径

**Glob 模式:** `**/SKILL.md` 和 `{skill,skills}/**/SKILL.md`

**命令:** 相同的目录结构但在 `commands/` 子目录下:
`/.slate/commands/`, `/.claude/commands/`, `/.agents/commands/`, `/.opencode/commands/`

**技能前置元数据:** 带有 `name` 和 `description` 字段的 YAML(根据 Slate 文档)。
两个字段均无文档记录的长度限制。

## 项目指令

Slate 读取 `CLAUDE.md` 和 `AGENTS.md` 作为项目指令。
两个字面字符串均在二进制文件中确认。现有 gstack 项目无需更改... CLAUDE.md 可直接使用。

## 配置

**配置文件:** `slate.json` / `slate.jsonc` (不是 opencode.json)

**配置选项(来自 Slate 文档):**
- `privacy` (布尔值) — 禁用遥测/日志记录
- 权限: 每个工具的 `allow`、`ask`、`deny` (`read`、`edit`、`bash`、`grep`、`webfetch`、`websearch`、`*`)
- 模型槽位: `models.main`、`models.subagent`、`models.search`、`models.reasoning`
- MCP 服务器: 本地或远程,带有自定义命令和标头
- 自定义命令: `/commands` 带模板

安装脚本不应创建 `slate.json`。用户配置自己的权限。

## CLI 标志(无头模式)

```
--stream-json / --output-format stream-json  — JSONL 输出,"与 Anthropic Claude Code SDK 兼容"
--dangerously-skip-permissions               — 绕过所有权限检查(CI/自动化)
--input-format stream-json                   — 程序化输入
-q                                           — 非交互模式
-w <dir>                                     — 工作空间目录
--output-format text                         — 纯文本输出(默认)
```

**Stream-JSON 格式:** Slate 文档声称"与 Anthropic Claude Code SDK 兼容"。
尚未经验证证实。鉴于 OpenCode 传承,可能与 Claude Code 的 NDJSON 事件模式匹配 (type: "assistant", type: "tool_result", type: "result")。

**需要验证:** 使用有效积分运行 `slate -q "hello" --stream-json` 并在构建会话运行器解析器之前捕获实际的 JSONL 事件。

## 环境变量(来自二进制字符串)

### Slate 特定
```
SLATE_API_KEY                              — API 密钥
SLATE_AGENT                                — 代理选择
SLATE_AUTO_SHARE                           — 自动共享设置
SLATE_CLIENT                               — 客户端标识符
SLATE_CONFIG                               — 配置覆盖
SLATE_CONFIG_CONTENT                       — 内联配置
SLATE_CONFIG_DIR                           — 配置目录
SLATE_DANGEROUSLY_SKIP_PERMISSIONS         — 绕过权限
SLATE_DIR                                  — 数据目录覆盖
SLATE_DISABLE_AUTOUPDATE                   — 禁用自动更新
SLATE_DISABLE_CLAUDE_CODE                  — 完全禁用 Claude Code 集成
SLATE_DISABLE_CLAUDE_CODE_PROMPT           — 禁用 Claude Code 提示加载
SLATE_DISABLE_CLAUDE_CODE_SKILLS           — 禁用 .claude/skills/ 加载
SLATE_DISABLE_DEFAULT_PLUGINS              — 禁用默认插件
SLATE_DISABLE_FILETIME_CHECK               — 禁用文件时间检查
SLATE_DISABLE_LSP_DOWNLOAD                 — 禁用 LSP 自动下载
SLATE_DISABLE_MODELS_FETCH                 — 禁用模型配置获取
SLATE_DISABLE_PROJECT_CONFIG               — 禁用项目级配置
SLATE_DISABLE_PRUNE                        — 禁用会话修剪
SLATE_DISABLE_TERMINAL_TITLE               — 禁用终端标题更新
SLATE_ENABLE_EXA                           — 启用 Exa 搜索
SLATE_ENABLE_EXPERIMENTAL_MODELS           — 启用实验性模型
SLATE_EXPERIMENTAL                         — 启用实验性功能
SLATE_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS — bash 超时覆盖
SLATE_EXPERIMENTAL_DISABLE_COPY_ON_SELECT  — 禁用选择时复制
SLATE_EXPERIMENTAL_DISABLE_FILEWATCHER     — 禁用文件监视器
SLATE_EXPERIMENTAL_EXA                     — Exa 搜索(备用标志)
SLATE_EXPERIMENTAL_FILEWATCHER             — 启用文件监视器
SLATE_EXPERIMENTAL_ICON_DISCOVERY          — 图标发现
SLATE_EXPERIMENTAL_LSP_TOOL               — LSP 工具
SLATE_EXPERIMENTAL_LSP_TY                 — LSP 类型检查
SLATE_EXPERIMENTAL_MARKDOWN               — markdown 模式
SLATE_EXPERIMENTAL_OUTPUT_TOKEN_MAX       — 输出令牌限制
SLATE_EXPERIMENTAL_OXFMT                  — oxfmt 集成
SLATE_EXPERIMENTAL_PLAN_MODE              — 计划模式
SLATE_FAKE_VCS                            — 用于测试的假 VCS
SLATE_GIT_BASH_PATH                       — git bash 路径(Windows)
SLATE_MODELS_URL                          — 模型配置 URL
SLATE_PERMISSION                          — 权限覆盖
SLATE_SERVER_PASSWORD                     — 服务器认证
SLATE_SERVER_USERNAME                     — 服务器认证
SLATE_TELEMETRY_DISABLED                  — 禁用遥测
SLATE_TEST_HOME                           — 测试主目录
SLATE_TOKEN_DIR                           — 令牌存储目录
```

### OpenCode 遗留(仍然有效)
```
OPENCODE_DISABLE_LSP_DOWNLOAD
OPENCODE_EXPERIMENTAL_DISABLE_FILEWATCHER
OPENCODE_EXPERIMENTAL_FILEWATCHER
OPENCODE_EXPERIMENTAL_ICON_DISCOVERY
OPENCODE_EXPERIMENTAL_LSP_TY
OPENCODE_EXPERIMENTAL_OXFMT
OPENCODE_FAKE_VCS
OPENCODE_GIT_BASH_PATH
OPENCODE_LIBC
OPENCODE_TERMINAL
```

### gstack 集成的关键环境变量

**`SLATE_DISABLE_CLAUDE_CODE_SKILLS`** — 设置后,`.claude/skills/` 加载被禁用。
这使得发布到 `.slate/skills/` 成为承重结构,而不仅仅是优化。
如果没有原生 `.slate/` 发布,当设置此标志时 gstack 技能会消失。

**`SLATE_TEST_HOME`** — 对 E2E 测试有用。可以将 Slate 的主目录重定向到隔离的临时目录,类似于 Codex 测试使用临时 HOME 的方式。

**`SLATE_DANGEROUSLY_SKIP_PERMISSIONS`** — 无头 E2E 测试所需。

## 模型引用(来自二进制文件)

```
anthropic/claude-sonnet-4.6
anthropic/claude-opus-4
anthropic/claude-haiku-4
anthropic/slate              — Slate 自己的模型路由
openai/gpt-5.3-codex
google/nano-banana
randomlabs/fast-default-alpha
```

## API 端点(来自二进制文件)

```
https://api.randomlabs.ai                          — 主 API
https://api.randomlabs.ai/exaproxy                 — Exa 搜索代理
https://agent-worker-prod.randomlabs.workers.dev   — 生产工作器
https://agent-worker-dev.randomlabs.workers.dev    — 开发工作器
https://dashboard.randomlabs.ai                    — 仪表板
https://docs.randomlabs.ai                         — 文档
https://randomlabs.ai/config.json                  — 远程配置
```

Brew tap: `anthropic/tap/slate` (值得注意: 在 Anthropic 的 tap 下,而非 Random Labs)

## npm 包结构

```
@randomlabs/slate (8.8 kB, 轻量启动器)
├── bin/slate           — Node.js 启动器(在 node_modules 中查找平台二进制文件)
├── bin/slate1          — Bun 启动器(相同逻辑, import.meta.filename)
├── postinstall.mjs     — 验证平台二进制文件存在,如需要则创建符号链接
└── package.json        — 声明所有平台的 optionalDependencies

平台包(每个 85MB):
├── @randomlabs/slate-darwin-arm64
├── @randomlabs/slate-darwin-x64
├── @randomlabs/slate-linux-arm64
├── @randomlabs/slate-linux-x64
├── @randomlabs/slate-linux-x64-musl
├── @randomlabs/slate-linux-arm64-musl
├── @randomlabs/slate-linux-x64-baseline
├── @randomlabs/slate-linux-x64-baseline-musl
├── @randomlabs/slate-darwin-x64-baseline
├── @randomlabs/slate-windows-x64
└── @randomlabs/slate-windows-x64-baseline
```

二进制覆盖: `SLATE_BIN_PATH` 环境变量跳过所有发现,直接运行指定的二进制文件。

## 目前已经可用的功能

gstack 技能已经通过 `.claude/skills/` 回退路径在 Slate 中工作。
基本功能无需更改。为 Claude Code 安装 gstack 并同时使用 Slate 的用户会发现他们的技能在两个代理中都可用。

## 一流支持增加的内容

1. **可靠性** — `.slate/skills/` 是 Slate 的最高优先级路径。不受 `SLATE_DISABLE_CLAUDE_CODE_SKILLS` 影响。
2. **优化的前置元数据** — 剥离 Slate 不使用的 Claude 特定字段(allowed-tools、hooks、version)。仅保留 `name` 和 `description`。
3. **安装脚本** — 自动检测 `slate` 二进制文件,将技能安装到 `~/.slate/skills/`。
4. **E2E 测试** — 验证技能在 Slate 直接调用时工作。

## 受阻于: 宿主配置重构

Codex 的外部审查确定,将 Slate 添加为第 4 个宿主(在 Claude、Codex、Factory 之后)是"路径别名的宿主爆炸"。当前架构具有:

- `type Host = 'claude' | 'codex' | 'factory'` 中的硬编码宿主名称
- `transformFrontmatter()` 中的每个宿主分支,具有近乎重复的逻辑
- `EXTERNAL_HOST_CONFIG` 中的每个宿主配置,具有相似的模式
- 安装脚本中的每个宿主函数 (`create_codex_runtime_root`、`link_codex_skill_dirs`)
- 在 `bin/gstack-platform-detect`、`bin/gstack-uninstall`、`bin/dev-setup` 中重复的宿主名称

添加 Slate 意味着再次复制所有这些模式。重构以使宿主数据驱动(配置对象而不是 if/else 分支)将使 Slate 集成变得简单,并使未来的宿主(任何新的 OpenCode 分支,任何新代理)零工作量。

### 计划中缺失的内容(由 Codex 识别)

- `lib/worktree.ts` 仅复制 `.agents/`,而不是 `.slate/` — 工作树中的 E2E 测试不会有 Slate 技能
- `bin/gstack-uninstall` 不知道 `.slate/`
- `bin/dev-setup` 不为贡献者开发模式连接 `.slate/`
- `bin/gstack-platform-detect` 不检测 Slate
- E2E 测试应设置 `SLATE_DISABLE_CLAUDE_CODE_SKILLS=1` 以证明 `.slate/` 路径实际工作(而不仅仅是回退到 `.claude/`)

## 会话运行器设计(稍后)

当 JSONL 格式得到验证后,会话运行器应该:

- 生成: `slate -q "<prompt>" --stream-json --dangerously-skip-permissions -w <dir>`
- 解析: Claude Code SDK 兼容的 NDJSON(假设,需要验证)
- 技能: 安装到测试夹具中的 `.slate/skills/`(而不是 `.claude/skills/`)
- 认证: 使用 `SLATE_API_KEY` 或现有的 `~/.slate/` 凭据
- 隔离: 使用 `SLATE_TEST_HOME` 进行主目录隔离
- 超时: 300s 默认(与 Codex 相同)

```typescript
export interface SlateResult {
  output: string;
  toolCalls: string[];
  tokens: number;
  exitCode: number;
  durationMs: number;
  sessionId: string | null;
  rawLines: string[];
  stderr: string;
}
```

## 文档参考

- Slate 文档: https://docs.randomlabs.ai
- 快速入门: https://docs.randomlabs.ai/en/getting-started/quickstart
- 技能: https://docs.randomlabs.ai/en/using-slate/skills
- 配置: https://docs.randomlabs.ai/en/using-slate/configuration
- 热键: https://docs.randomlabs.ai/en/using-slate/hotkey_reference