# 架构

本文档解释 gstack **为什么**这样构建。关于设置和命令,请参阅 CLAUDE.md。关于贡献,请参阅 CONTRIBUTING.md。

## 核心理念

gstack 为 Claude Code 提供了一个持久化浏览器和一组有主见的工作流技能。浏览器是难点——其他一切都是 Markdown。

关键洞察:与浏览器交互的 AI 代理需要**亚秒级延迟**和**持久化状态**。如果每个命令都冷启动浏览器,每次工具调用要等待 3-5 秒。如果浏览器在命令之间死掉,你会丢失 cookies、标签页和登录会话。所以 gstack 运行一个长期存活的 Chromium 守护进程,CLI 通过 localhost HTTP 与之通信。

```
Claude Code                     gstack
─────────                      ──────
                               ┌──────────────────────┐
  工具调用: $B snapshot -i      │  CLI (编译的二进制文件) │
  ─────────────────────────→   │  • 读取状态文件        │
                               │  • POST /command      │
                               │    到 localhost:PORT   │
                               └──────────┬───────────┘
                                          │ HTTP
                               ┌──────────▼───────────┐
                               │  Server (Bun.serve)   │
                               │  • 分发命令            │
                               │  • 与 Chromium 通信    │
                               │  • 返回纯文本          │
                               └──────────┬───────────┘
                                          │ CDP
                               ┌──────────▼───────────┐
                               │  Chromium (headless)   │
                               │  • 持久化标签页        │
                               │  • cookies 保留        │
                               │  • 30分钟空闲超时      │
                               └───────────────────────┘
```

首次调用启动所有组件(约 3 秒)。之后每次调用:约 100-200 毫秒。

## 为什么选择 Bun

Node.js 也能工作。但 Bun 在这里更好,原因有三:

1. **编译的二进制文件。** `bun build --compile` 生成单个约 58MB 的可执行文件。运行时无需 `node_modules`,无需 `npx`,无需 PATH 配置。二进制文件直接运行。这很重要,因为 gstack 安装到 `~/.claude/skills/`,用户不希望在那里管理 Node.js 项目。

2. **原生 SQLite。** Cookie 解密直接读取 Chromium 的 SQLite cookie 数据库。Bun 内置 `new Database()` ——无需 `better-sqlite3`,无需原生插件编译,无需 gyp。少了一个在不同机器上出问题的环节。

3. **原生 TypeScript。** 开发时服务器以 `bun run server.ts` 运行。无需编译步骤,无需 `ts-node`,无需调试 source maps。编译的二进制文件用于部署;源文件用于开发。

4. **内置 HTTP 服务器。** `Bun.serve()` 快速、简单,不需要 Express 或 Fastify。服务器总共处理约 10 个路由。框架会是额外开销。

瓶颈始终是 Chromium,而不是 CLI 或服务器。Bun 的启动速度(编译二进制约 1 毫秒 vs Node 约 100 毫秒)很好但不是我们选择它的原因。编译的二进制文件和原生 SQLite 才是。

## 守护进程模型

### 为什么不是每个命令启动一个浏览器?

Playwright 可以在约 2-3 秒内启动 Chromium。对于单次截图,这没问题。对于有 20 多个命令的 QA 会话,就是 40 多秒的浏览器启动开销。更糟的是:命令之间会丢失所有状态。Cookies、localStorage、登录会话、打开的标签页——全部消失。

守护进程模型意味着:

- **持久化状态。** 登录一次,保持登录。打开标签页,它保持打开。localStorage 在命令之间持久化。
- **亚秒级命令。** 首次调用后,每个命令只是一个 HTTP POST。包括 Chromium 的工作在内约 100-200 毫秒往返。
- **自动生命周期。** 服务器在首次使用时自动启动,空闲 30 分钟后自动关闭。无需进程管理。

### 状态文件

服务器写入 `.gstack/browse.json`(通过 tmp + rename 原子写入,模式 0o600):

```json
{ "pid": 12345, "port": 34567, "token": "uuid-v4", "startedAt": "...", "binaryVersion": "abc123" }
```

CLI 读取此文件以找到服务器。如果文件缺失或服务器未通过 HTTP 健康检查,CLI 会生成新服务器。在 Windows 上,基于 PID 的进程检测在 Bun 二进制文件中不可靠,因此健康检查(GET /health)是所有平台上的主要存活信号。

### 端口选择

10000-60000 之间的随机端口(冲突时最多重试 5 次)。这意味着 10 个 Conductor 工作区可以各自运行自己的 browse 守护进程,零配置且零端口冲突。旧方法(扫描 9400-9409)在多工作区设置中经常出问题。

### 版本自动重启

构建时将 `git rev-parse HEAD` 写入 `browse/dist/.version`。每次 CLI 调用时,如果二进制文件的版本与运行中服务器的 `binaryVersion` 不匹配,CLI 会杀死旧服务器并启动新服务器。这完全防止了"陈旧二进制文件"类错误——重新构建二进制文件,下一个命令会自动使用它。

## 安全模型

### 仅限 localhost

HTTP 服务器绑定到 `127.0.0.1`,而不是 `0.0.0.0`。无法从网络访问。

### 双监听器隧道架构(v1.6.0.0)

当用户运行 `pair-agent --client` 时,守护进程启动 ngrok 隧道,以便远程配对代理可以驱动浏览器。将完整的守护进程表面暴露到互联网(即使在随机 ngrok 子域后面)意味着 `/health` 在任何 Origin 欺骗时泄露根令牌,而 `/cookie-picker` 将令牌嵌入到任何调用者都可以获取的 HTML 中。

修复方案是**两个 HTTP 监听器**,而不是一个:

- **本地监听器**(`127.0.0.1:LOCAL_PORT`)——始终绑定。提供引导(`/health` 带令牌传递)、`/cookie-picker`、`/inspector/*`、`/welcome`、`/refs`、侧边栏代理 API 和完整命令表面。从不转发。
- **隧道监听器**(`127.0.0.1:TUNNEL_PORT`)——在 `/tunnel/start` 时延迟绑定,在 `/tunnel/stop` 时拆除。提供锁定的允许列表:`/connect`(配对仪式,未认证 + 速率限制)、`/command`(仅限作用域令牌,进一步限制为浏览器驱动命令允许列表)和 `/sidebar-chat`。其他一切返回 404。

ngrok 仅转发隧道端口。安全属性来自**物理端口分离**:隧道调用者无法访问 `/health` 或 `/cookie-picker`,因为这些路径在该 TCP 套接字上不存在。头部推断(检查 `x-forwarded-for`,检查 origin)不可靠(ngrok 头部行为会变化;本地代理可以添加这些头部);套接字分离则不会。

| 端点 | 本地监听器 | 隧道监听器 | 备注 |
|---|---|---|---|
| `GET /health` | 公开(除非 headed/extension 否则无令牌) | 404 | 扩展的令牌引导仅在本地发生 |
| `GET /connect` | 公开(`{alive:true}`) | 公开(`{alive:true}`) | 隧道存活性探测路径 |
| `POST /connect` | 公开(速率限制 300/分钟) | 公开(速率限制) | pair-agent 的设置密钥交换 |
| `POST /command` | 认证(Bearer 根令牌或作用域令牌) | 认证(仅作用域令牌,允许列表命令) | 隧道上的根令牌 = 403 |
| `POST /sidebar-chat` | 认证 | 认证 | 让远程代理发布到本地侧边栏 |
| `POST /pair` | 仅限根令牌 | 404 | 配对铸造——本地操作员操作 |
| `POST /tunnel/{start,stop}` | 仅限根令牌 | 404 | 守护进程配置 |
| `POST /token`, `DELETE /token/:id` | 仅限根令牌 | 404 | 作用域令牌铸造/撤销 |
| `GET /cookie-picker`, `GET /cookie-picker/*` | 公开 UI,认证 API | 404 | 仅限本地——读取本地浏览器数据库 |
| `GET /inspector`, `/inspector/events` 等 | 认证 | 404 | 扩展回调,仅限本地 |
| `GET /welcome` | 公开 | 404 | GStack Browser 着陆页,仅限本地 |
| `GET /refs` | 认证 | 404 | Ref 映射——内部状态 |
| `GET /activity/stream` | Bearer 或 HttpOnly `gstack_sse` cookie | 404 | SSE。不再接受 ?token= 查询参数 |
| `GET /inspector/events` | Bearer 或 HttpOnly `gstack_sse` cookie | 404 | SSE。与 /activity/stream 相同的 cookie |
| `POST /sse-session` | 认证(Bearer) | 404 | 铸造仅查看的 30 分钟 SSE 会话 cookie |

**隧道表面拒绝日志。** 隧道监听器上的每次拒绝(`path_not_on_tunnel`、`root_token_on_tunnel`、`missing_scoped_token`、`disallowed_command:*`)都会异步记录到 `~/.gstack/security/attempts.jsonl`,包含时间戳、源 IP(来自 `x-forwarded-for`)、路径和方法。全局速率限制为 60 次写入/分钟以防止日志洪水 DoS。与提示注入扫描器共享尝试日志。

**SSE 会话 cookies。** EventSource 无法发送 Authorization 头部,因此扩展在引导时使用根 Bearer POST 一次 `/sse-session` 并接收 30 分钟仅查看 cookie(`gstack_sse`,HttpOnly,SameSite=Strict)。该 cookie 仅对 `/activity/stream` 和 `/inspector/events` 有效——它不是作用域令牌,不能用于 `/command`。作用域隔离由模块边界强制执行:`sse-session-cookie.ts` 没有从 `token-registry.ts` 导入。

**本波次非目标**(跟踪为 #1136):cookie-import-browser 路径使用 `--remote-debugging-port=<random>` 启动 Chrome。在 Windows 上使用 App-Bound Encryption v20 时,同用户本地进程可以连接到该端口并泄露解密的 v20 cookies——相对于直接读取 SQLite 数据库(没有 DPAPI 上下文无法解密 v20)的提升路径。修复方向是使用 `--remote-debugging-pipe` 而不是 TCP;需要重构 CDP 客户端。

### Bearer 令牌认证

每个服务器会话生成一个随机 UUID 令牌,写入状态文件,模式为 0o600(仅所有者可读)。每个改变浏览器状态的 HTTP 请求必须包含 `Authorization: Bearer <token>`。如果令牌不匹配,服务器返回 401。

这防止同一台机器上的其他进程与你的 browse 服务器通信。Cookie 选择器 UI(`/cookie-picker`)和健康检查(`/health`)在本地监听器上豁免——它们绑定到 127.0.0.1 且不执行命令。在隧道监听器上除了 `/connect` 外没有豁免。

### Cookie 安全

Cookies 是 gstack 处理的最敏感数据。设计如下:

1. **钥匙串访问需要用户批准。** 每个浏览器的首次 cookie 导入会触发 macOS 钥匙串对话框。用户必须点击"允许"或"始终允许"。gstack 从不静默访问凭据。

2. **解密在进程内发生。** Cookie 值在内存中解密(PBKDF2 + AES-128-CBC),加载到 Playwright 上下文中,从不以明文写入磁盘。Cookie 选择器 UI 从不显示 cookie 值——只显示域名和计数。

3. **数据库是只读的。** gstack 将 Chromium cookie 数据库复制到临时文件(以避免与运行中浏览器的 SQLite 锁冲突)并以只读方式打开。它从不修改你真实浏览器的 cookie 数据库。

4. **密钥缓存是每会话的。** 钥匙串密码 + 派生的 AES 密钥在内存中缓存,持续服务器的生命周期。当服务器关闭(空闲超时或显式停止)时,缓存消失。

5. **日志中没有 cookie 值。** 控制台、网络和对话框日志从不包含 cookie 值。`cookies` 命令输出 cookie 元数据(域、名称、过期时间),但值被截断。

### Shell 注入防护

浏览器注册表(Comet、Chrome、Arc、Brave、Edge)是硬编码的。数据库路径从已知常量构造,从不从用户输入构造。钥匙串访问使用 `Bun.spawn()` 和显式参数数组,而不是 shell 字符串插值。

### 提示注入防御(侧边栏代理)

Chrome 侧边栏代理有工具(Bash、Read、Glob、Grep、WebFetch)并读取恶意网页,因此它是 gstack 中最容易受到提示注入攻击的部分。防御是分层的,而不是单点的。

1. **L1-L3 内容安全(`browse/src/content-security.ts`)。** 在每个页面内容命令和每个工具输出上运行:数据标记、隐藏元素剥离、ARIA 正则表达式、URL 黑名单和信任边界包络包装器。在服务器和代理上都应用。

2. **L4 ML 分类器——TestSavantAI(`browse/src/security-classifier.ts`)。** 一个 22MB 的 BERT-small ONNX 模型(int8 量化),与代理捆绑。本地运行,无网络。在 Claude 看到之前扫描每条用户消息和每个 Read/Glob/Grep/WebFetch 工具输出。通过 `GSTACK_SECURITY_ENSEMBLE=deberta` 选择加入 721MB DeBERTa-v3 集成。

3. **L4b 对话记录分类器。** 一个 Claude Haiku 通道,查看完整对话形状(用户消息、工具调用、工具输出),而不仅仅是文本。由 `LOG_ONLY: 0.40` 控制,因此大多数干净流量跳过付费调用。

4. **L5 金丝雀令牌(`browse/src/security.ts`)。** 会话开始时注入系统提示的随机令牌。跨 `text_delta` 和 `input_json_delta` 流的滚动缓冲区检测在 Claude 的输出、工具参数、URL 或文件写入中的任何地方出现令牌时捕获它。确定性 BLOCK——如果令牌泄露,攻击者说服 Claude 揭示系统提示,会话结束。

5. **L6 集成组合器(`combineVerdict`)。** BLOCK 需要两个 ML 分类器在 >= `WARN`(0.60)时达成一致,而不是单个高置信度命中。这是 Stack Overflow 指令编写误报缓解。在工具输出扫描时,单层高置信度直接 BLOCK——内容不是用户创作的,因此不存在误报问题。

**关键约束:** `security-classifier.ts` 仅在侧边栏代理进程中运行,从不在编译的 browse 二进制文件中运行。`@huggingface/transformers` v4 需要 `onnxruntime-node`,它在 Bun compile 的临时提取目录中 `dlopen` 失败。只有纯字符串部分(金丝雀注入/检查、判决组合器、攻击日志、状态)在 `security.ts` 中,可以安全地从 `server.ts` 导入。

**环境旋钮:** `GSTACK_SECURITY_OFF=1` 是真正的终止开关(跳过 ML 扫描,金丝雀仍然注入)。模型缓存在 `~/.gstack/models/testsavant-small/`(112MB,首次运行)和 `~/.gstack/models/deberta-v3-injection/`(721MB,仅选择加入)。攻击日志在 `~/.gstack/security/attempts.jsonl`(加盐 sha256 + 域,在 10MB 时轮换,5 代)。每设备盐在 `~/.gstack/security/device-salt`(0600),在进程中缓存以在 FS 不可写环境中存活。

**可见性。** 侧边栏标题显示通过 `/sidebar-chat` 轮询的盾牌图标(绿色/琥珀色/红色)。在金丝雀泄露或 BLOCK 判决时出现居中横幅,显示确切的层分数。`bin/gstack-security-dashboard` 聚合本地尝试;`supabase/functions/community-pulse` 聚合跨用户的选择加入社区遥测。

## Ref 系统

Refs(`@e1`、`@e2`、`@c1`)是代理在不编写 CSS 选择器或 XPath 的情况下寻址页面元素的方式。

### 工作原理

```
1. 代理运行: $B snapshot -i
2. 服务器调用 Playwright 的 page.accessibility.snapshot()
3. 解析器遍历 ARIA 树,分配顺序 refs: @e1, @e2, @e3...
4. 对于每个 ref,构建 Playwright Locator: getByRole(role, { name }).nth(index)
5. 在 BrowserManager 实例上存储 Map<string, RefEntry>(role + name + Locator)
6. 将带注释的树作为纯文本返回

稍后:
7. 代理运行: $B click @e3
8. 服务器解析 @e3 → Locator → locator.click()
```

### 为什么是 Locators,而不是 DOM 变更

显而易见的方法是将 `data-ref="@e1"` 属性注入 DOM。这在以下情况下会失败:

- **CSP(内容安全策略)。** 许多生产站点阻止脚本的 DOM 修改。
- **React/Vue/Svelte 水合。** 框架协调可以剥离注入的属性。
- **Shadow DOM。** 无法从外部到达 shadow roots 内部。

Playwright Locators 在 DOM 外部。它们使用可访问性树(Chromium 内部维护)和 `getByRole()` 查询。无 DOM 变更,无 CSP 问题,无框架冲突。

### Ref 生命周期

Refs 在导航时清除(主框架上的 `framenavigated` 事件)。这是正确的——导航后,所有 locators 都是陈旧的。代理必须再次运行 `snapshot` 以获取新的 refs。这是设计使然:陈旧的 refs 应该大声失败,而不是点击错误的元素。

### Ref 陈旧性检测

SPA 可以在不触发 `framenavigated` 的情况下改变 DOM(例如 React 路由器转换、标签切换、模态框打开)。这使得 refs 陈旧,即使页面 URL 没有改变。为了捕获这一点,`resolveRef()` 在使用任何 ref 之前执行异步 `count()` 检查:

```
resolveRef(@e3) → entry = refMap.get("e3")
                → count = await entry.locator.count()
                → if count === 0: throw "Ref @e3 已陈旧——元素不再存在。运行 'snapshot' 以获取新的 refs。"
                → if count > 0: return { locator }
```

这会快速失败(约 5 毫秒开销),而不是让 Playwright 的 30 秒操作超时在缺失元素上过期。`RefEntry` 在 Locator 旁边存储 `role` 和 `name` 元数据,因此错误消息可以告诉代理元素是什么。

### 光标交互 refs(@c)

`-C` 标志查找可点击但不在 ARIA 树中的元素——用 `cursor: pointer` 样式化的东西、带有 `onclick` 属性的元素或自定义 `tabindex`。这些在单独的命名空间中获得 `@c1`、`@c2` refs。这捕获了框架渲染为 `<div>` 但实际上是按钮的自定义组件。

## 日志架构

三个环形缓冲区(每个 50,000 条目,O(1) 推送):

```
浏览器事件 → CircularBuffer(内存中) → 异步刷新到 .gstack/*.log
```

控制台消息、网络请求和对话框事件各有自己的缓冲区。刷新每 1 秒发生一次——服务器仅追加自上次刷新以来的新条目。这意味着:

- HTTP 请求处理从不被磁盘 I/O 阻塞
- 日志在服务器崩溃后存活(最多 1 秒的数据丢失)
- 内存是有界的(50K 条目 × 3 个缓冲区)
- 磁盘文件是仅追加的,可被外部工具读取

`console`、`network` 和 `dialog` 命令从内存缓冲区读取,而不是磁盘。磁盘文件用于事后调试。

## SKILL.md 模板系统

### 问题

SKILL.md 文件告诉 Claude 如何使用 browse 命令。如果文档列出了不存在的标志,或遗漏了添加的命令,代理会遇到错误。手动维护的文档总是与代码脱节。

### 解决方案

```
SKILL.md.tmpl          (人工编写的散文 + 占位符)
       ↓
gen-skill-docs.ts      (读取源代码元数据)
       ↓
SKILL.md               (已提交,自动生成的部分)
```

模板包含需要人工判断的工作流、提示和示例。占位符在构建时从源代码填充:

| 占位符 | 来源 | 生成内容 |
|-------------|--------|-------------------|
| `{{COMMAND_REFERENCE}}` | `commands.ts` | 分类命令表 |
| `{{SNAPSHOT_FLAGS}}` | `snapshot.ts` | 带示例的标志参考 |
| `{{PREAMBLE}}` | `gen-skill-docs.ts` | 启动块:更新检查、会话跟踪、贡献者模式、AskUserQuestion 格式 |
| `{{BROWSE_SETUP}}` | `gen-skill-docs.ts` | 二进制发现 + 设置说明 |
| `{{BASE_BRANCH_DETECT}}` | `gen-skill-docs.ts` | PR 目标技能的动态基础分支检测(ship、review、qa、plan-ceo-review) |
| `{{QA_METHODOLOGY}}` | `gen-skill-docs.ts` | /qa 和 /qa-only 的共享 QA 方法论块 |
| `{{DESIGN_METHODOLOGY}}` | `gen-skill-docs.ts` | /plan-design-review 和 /design-review 的共享设计审计方法论 |
| `{{REVIEW_DASHBOARD}}` | `gen-skill-docs.ts` | /ship 起飞前的审查就绪仪表板 |
| `{{TEST_BOOTSTRAP}}` | `gen-skill-docs.ts` | /qa、/ship、/design-review 的测试框架检测、引导、CI/CD 设置 |
| `{{CODEX_PLAN_REVIEW}}` | `gen-skill-docs.ts` | /plan-ceo-review 和 /plan-eng-review 的可选跨模型计划审查(Codex 或 Claude 子代理回退) |
| `{{DESIGN_SETUP}}` | `resolvers/design.ts` | `$D` 设计二进制文件的发现模式,镜像 `{{BROWSE_SETUP}}` |
| `{{DESIGN_SHOTGUN_LOOP}}` | `resolvers/design.ts` | /design-shotgun、/plan-design-review、/design-consultation 的共享比较板反馈循环 |
| `{{UX_PRINCIPLES}}` | `resolvers/design.ts` | /design-html、/design-shotgun、/design-review、/plan-design-review 的用户行为基础(扫描、满意、善意储备、主干测试) |
| `{{GBRAIN_CONTEXT_LOAD}}` | `resolvers/gbrain.ts` | 带关键词提取、健康意识和数据研究路由的大脑优先上下文搜索。注入到 10 个大脑感知技能中。在非大脑主机上抑制。 |
| `{{GBRAIN_SAVE_RESULTS}}` | `resolvers/gbrain.ts` | 带实体丰富、节流处理和每技能保存说明的技能后大脑持久化。8 种技能特定保存格式。 |

这在结构上是合理的——如果命令存在于代码中,它就会出现在文档中。如果它不存在,它就不能出现。

### 前言

每个技能都以 `{{PREAMBLE}}` 块开始,在技能自己的逻辑之前运行。它在单个 bash 命令中处理五件事:

1. **更新检查**——调用 `gstack-update-check`,报告是否有可用升级。
2. **会话跟踪**——触摸 `~/.gstack/sessions/$PPID` 并计算活动会话(最近 2 小时内修改的文件)。当 3 个以上会话运行时,所有技能进入"ELI16 模式"——每个问题都重新为用户提供上下文,因为他