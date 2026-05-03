# 侧边栏流程

GStack Browser 侧边栏的实际工作原理。在修改 `sidepanel.js`、`background.js`、`content.js`、`terminal-agent.ts` 或侧边栏相关服务器端点之前,请先阅读本文档。

侧边栏有一个主要界面 — **Terminal** 面板,一个交互式的 `claude` PTY。Activity / Refs / Inspector 作为调试覆盖层保留在页脚的 `debug` 开关后面。聊天队列路径(一次性 `claude -p`,sidebar-agent.ts)在 PTY 验证通过后被移除 — Terminal 面板的功能严格来说更强大。

## 组件

```
┌─────────────────┐     ┌──────────────┐     ┌──────────────────┐
│  sidepanel.js + │────▶│  server.ts   │────▶│terminal-agent.ts │
│  -terminal.js   │     │  (已编译)     │     │  (未编译)         │
│  (xterm.js)     │     │              │     │  PTY 监听器      │
└─────────────────┘     └──────────────┘     └──────────────────┘
        ▲                       │                      │
        │  ws://127.0.0.1:<termPort>/ws (Sec-WebSocket-Protocol 认证)
        └───────────────────────┼──────────────────────▶│ Bun.spawn(claude)
                                │                      │  terminal: {data}
                                │                      ▼
                                │              ┌──────────────────┐
                                │              │  claude PTY      │
                                │              └──────────────────┘
            POST /pty-session   │
            (Bearer AUTH_TOKEN) │
                                ▼
                       ┌──────────────────┐
                       │ pty-session-     │
                       │ cookie.ts        │
                       │ (内存中的令牌    │
                       │  注册表)         │
                       └──────────────────┘
                                │
                                │ POST /internal/grant (回环)
                                ▼
                       ┌──────────────────┐
                       │  validTokens Set │
                       │  在 agent 内存中 │
                       └──────────────────┘
```

已编译的浏览服务器无法 `posix_spawn` 外部可执行文件 — `terminal-agent.ts` 作为单独的未编译 `bun run` 进程运行,并拥有 `claude` 子进程。

## 启动 + 首次按键时间线

```
T+0ms     CLI 运行 `$B connect`
            ├── 服务器启动(已编译)
            └── 通过 `bun run` 生成 terminal-agent.ts

T+500ms   terminal-agent.ts 启动
            ├── Bun.serve 在 127.0.0.1:0 (随机端口)
            ├── 写入 <stateDir>/terminal-port (服务器读取它用于 /health)
            ├── 写入 <stateDir>/terminal-internal-token (回环握手)
            └── 探测 claude → 写入 claude-available.json

T+1-3s    扩展加载,侧边栏打开
            ├── sidepanel-terminal.js: setState(IDLE),显示 "Starting Claude Code..."
            └── tryAutoConnect() 轮询直到 window.gstackServerPort + token 被设置

T+ready   tryAutoConnect 调用 connect()
            ├── POST /pty-session (Authorization: Bearer AUTH_TOKEN)
            │   └── 服务器生成会话令牌,向 agent 发送 /internal/grant
            │   └── 响应 {terminalPort, ptySessionToken}
            ├── GET /claude-available (预检)
            ├── new WebSocket(`ws://127.0.0.1:<terminalPort>/ws`,
            │                 [`gstack-pty.<token>`])
            │   └── 浏览器发送 Sec-WebSocket-Protocol + Origin
            │   └── Agent 在升级之前验证 Origin 和 token
            │   └── Agent 回显协议(必需 — 浏览器
            │       没有它会关闭连接)
            ├── 打开时:发送 {type:"resize"} 然后发送单个 \n 字节
            └── Agent 消息处理器看到字节 → spawnClaude()
```

## 认证:WebSocket 无法发送 Authorization 头

浏览器 WebSocket 客户端无法设置 `Authorization`。它们可以通过 `new WebSocket(url, protocols)` 的第二个参数设置 `Sec-WebSocket-Protocol`。我们利用这一点:

1. `POST /pty-session` (auth: Bearer AUTH_TOKEN) → 服务器生成一个短期会话令牌,通过回环将其推送到 agent,在 JSON 响应体中返回它。
2. 扩展调用 `new WebSocket(url, ['gstack-pty.<token>'])`。
3. Agent 读取 `Sec-WebSocket-Protocol`,去除 `gstack-pty.`,根据 `validTokens` 验证,回显协议。回显是强制性的 — 没有它,Chromium 会在收到升级响应时关闭连接。

对于非浏览器调用者(curl、集成测试),还会返回 `Set-Cookie: gstack_pty=...` 头。Cookie 路径是原始的 v1 设计,但 `SameSite=Strict` cookie 无法在从 chrome-extension 源的 server.ts:34567 → agent:<random> 的跨端口跳转中存活。协议令牌路径是浏览器实际使用的。

### 双令牌模型

| 令牌 | 存储位置 | 用途 | 生命周期 |
|-------|----------|----------|----------|
| `AUTH_TOKEN` | `<stateDir>/browse.json`;server.ts 内存中 | `/pty-session` POST(生成 cookie + token) | 服务器生命周期 |
| `gstack-pty.<...>` (Sec-WebSocket-Protocol) | 仅浏览器内存;agent `validTokens` Set | `/ws` 升级认证 | 30 分钟,WS 关闭时自动撤销 |
| `INTERNAL_TOKEN` | `<stateDir>/terminal-internal-token`;agent 内存中 | server → agent 回环 `/internal/grant` | agent 生命周期 |

`AUTH_TOKEN` **永远不会**直接对 `/ws` 有效。会话令牌**永远不会**对 `/pty-session` 或 `/command` 有效。严格分离防止 SSE 或页面内容令牌泄漏升级为 shell 访问。

## 威胁模型

Terminal 面板**故意绕过提示注入安全栈** — 用户直接向 claude 输入,循环中没有不受信任的页面内容。信任源是键盘,与任何本地终端相同。

该信任假设依赖于三个传输保证:

1. **仅本地监听器。** terminal-agent.ts 仅绑定 `127.0.0.1`。双监听器隧道界面(server.ts `TUNNEL_PATHS`)不包括 `/pty-session` 或 `/terminal/*`,因此隧道默认拒绝返回 404。
2. **Origin 门控。** `/ws` 升级需要 `Origin: chrome-extension://<id>`。localhost 网页无法对 shell 发起跨站 WebSocket 劫持,因为其 Origin 是常规的 `http(s)://...`。
3. **会话令牌认证。** 仅由经过身份验证的 `/pty-session` POST 生成,作用域限定为一个 WS,关闭时自动撤销。

丢失这三个中的任何一个,整个标签页都会变得不安全。

## 生命周期

- **主动自动连接。** 侧边栏打开 → tryAutoConnect 轮询引导全局变量,并在它们被设置后立即连接。无需按键。
- **每个 WS 一个 PTY。** 关闭 WebSocket 会 SIGINT claude,然后在 3 秒后 SIGKILL。会话令牌被撤销,因此被盗令牌无法重放。
- **关闭时不自动重连。** 用户看到"会话已结束,点击开始新会话。"自动重连会在每次重新加载时消耗一个新的 claude 会话。v1.1 可能会添加基于标签页/会话 id 的会话恢复(参见 TODOS)。
- **随时手动重启。** 一个 `↻ Restart` 按钮位于始终可见的终端工具栏中 — 在会话中工作,不仅仅是从 ENDED 状态。

## 快速操作工具栏

三个浏览器操作按钮位于 Terminal 面板顶部的 Restart 按钮旁边:

| 按钮 | 行为 |
|--------|----------|
| 🧹 Cleanup | `window.gstackInjectToTerminal(prompt)` — 将"删除广告/横幅"指令注入到活动 PTY 中。终端中的 claude 看到它并执行操作。 |
| 📸 Screenshot | `POST /command screenshot` — 直接调用 browse-server,不涉及 PTY。 |
| 🍪 Cookies | 导航到 `/cookie-picker` 页面。 |

Inspector 的"Send to Code"按钮使用相同的 `gstackInjectToTerminal` 路径将 CSS inspector 数据转发到 claude。

## 调试界面(Activity / Refs / Inspector)

位于页脚的 `debug` 开关后面。SSE 驱动,独立于 Terminal 面板:

- **Activity** — 通过 `/activity/stream` SSE 流式传输每个浏览命令。
- **Refs** — REST:`GET /refs` — 当前页面的 `@ref` 元素标签。
- **Inspector** — 基于 CDP 的元素选择器;在 `/inspector/events` 上的 SSE。

当调试条关闭时,Terminal 面板重新变为可见。当其容器从 `display:none` 翻转到 `display:flex` 时,xterm.js 不会自动重绘,因此 sidepanel-terminal.js 在 `#tab-terminal` 的 class 属性上运行 `MutationObserver`,并在 `.active` 返回时强制进行适配和刷新。

## 文件

| 组件 | 文件 | 运行环境 |
|-----------|------|---------|
| 侧边栏 UI 外壳 | `extension/sidepanel.html` + `sidepanel.js` + `sidepanel.css` | Chrome 侧边面板 |
| Terminal UI | `extension/sidepanel-terminal.js` + `extension/lib/xterm.js` | Chrome 侧边面板 |
| Service worker | `extension/background.js` | Chrome 后台 |
| Content script | `extension/content.js` | 页面上下文 |
| HTTP 服务器 | `browse/src/server.ts` | Bun(已编译二进制) |
| PTY agent | `browse/src/terminal-agent.ts` | Bun(未编译) |
| PTY 令牌存储 | `browse/src/pty-session-cookie.ts` | Bun(已编译,在 server.ts 中) |
| CLI 入口 | `browse/src/cli.ts` | Bun(已编译二进制) |
| 状态文件 | `<stateDir>/browse.json` | 文件系统 |
| Terminal 端口 | `<stateDir>/terminal-port` | 文件系统 |
| 内部令牌 | `<stateDir>/terminal-internal-token` | 文件系统 |
| Claude 探测 | `<stateDir>/claude-available.json` | 文件系统 |
| 活动标签页 | `<stateDir>/active-tab.json` | 文件系统(claude 读取) |