# 远程浏览器访问 — 如何与 GStack 浏览器配对

GStack 浏览器服务器可以与任何能够发起 HTTP 请求的 AI 代理共享。
代理获得对真实 Chromium 浏览器的受限访问权限:导航页面、读取内容、
点击元素、填写表单、截图。每个代理都有自己的标签页。

本文档是远程代理的参考文档。快速入门说明由 `$B pair-agent` 生成,其中包含实际的凭证信息。

## 架构

```
您的机器                              远程代理
─────────────                         ────────────
GStack 浏览器服务器                    任何 AI 代理
  ├── Chromium (Playwright)           (OpenClaw, Hermes, Codex 等)
  ├── 本地监听器  127.0.0.1:LOCAL         │
  │    (引导、CLI、侧边栏、cookies)         │
  ├── 隧道监听器 127.0.0.1:TUNNEL ◄───────┤
  │    (仅限 pair-agent: /connect, /command,   │
  │     /sidebar-chat — 锁定的白名单)      │
  ├── ngrok 隧道 (仅转发隧道端口)          │
  │     https://xxx.ngrok.dev ─────────────────┘
  └── 令牌注册表
        ├── 根令牌 (仅限本地监听器)
        ├── 设置密钥 (5 分钟,一次性)
        ├── 会话令牌 (24小时,受限范围)
        └── SSE 会话 cookies (30 分钟,流范围)
```

### 双监听器架构 (v1.6.0.0)

守护进程绑定两个 HTTP 套接字。**本地监听器**仅向 127.0.0.1 提供完整的命令接口,永远不会被转发。**隧道监听器**在 `/tunnel/start` 时延迟绑定(并在 `/tunnel/stop` 时拆除),具有锁定的路径白名单。ngrok 仅转发隧道端口。

偶然访问您的 ngrok URL 的调用者无法访问 `/health`、`/cookie-picker`、`/inspector/*` 或 `/welcome` — 这些路径在该 TCP 套接字上不存在。通过隧道发送的根令牌会收到 403。隧道监听器仅接受 `/connect`、`/command`(带有受限令牌 + 26 个浏览器驱动命令白名单)和 `/sidebar-chat`。

完整的端点表请参见 [ARCHITECTURE.md](../ARCHITECTURE.md#dual-listener-tunnel-architecture-v1600)。

## 连接流程

1. **用户运行** `$B pair-agent`(或在 Claude Code 中运行 `/pair-agent`)
2. **服务器创建**一次性设置密钥(5 分钟后过期)
3. **用户复制**指令块到另一个代理的聊天中
4. **远程代理运行** `POST /connect` 并携带设置密钥
5. **服务器返回**受限会话令牌(默认 24 小时)
6. **远程代理创建**自己的标签页,通过 `POST /command` 使用 `newtab`
7. **远程代理浏览**,使用 `POST /command` 并携带会话令牌 + tabId

## API 参考

### 身份验证

所有命令端点都需要 Bearer 令牌:

```
Authorization: Bearer gsk_sess_...
```

`/connect` 无需身份验证(有速率限制)— 这是远程代理用设置密钥交换受限会话令牌的方式。`/health` 在本地监听器上无需身份验证(引导),但在隧道监听器上不存在(404)。

SSE 端点(`/activity/stream`、`/inspector/events`)接受 Bearer 令牌或 HttpOnly `gstack_sse` cookie(通过 `POST /sse-session` 生成,30 分钟 TTL,仅限流范围 — 不能用于 `/command`)。从 v1.6.0.0 开始,不再接受 `?token=<ROOT>` 查询字符串身份验证。

### 端点

#### POST /connect
用设置密钥交换会话令牌。无需身份验证。速率限制为 300 次/分钟(防洪 — 设置密钥是 24 个随机字节,无法暴力破解)。

```json
请求:  {"setup_key": "gsk_setup_..."}
响应: {"token": "gsk_sess_...", "expires": "ISO8601", "scopes": ["read","write"], "agent": "agent-name"}
```

#### POST /command
发送浏览器命令。需要 Bearer 身份验证。

```json
请求:  {"command": "goto", "args": ["https://example.com"], "tabId": 1}
响应: (命令的纯文本结果)
```

#### GET /health
服务器状态。无需身份验证。返回状态、标签页、模式、运行时间。

### 命令

#### 导航
| 命令 | 参数 | 描述 |
|---------|------|-------------|
| `goto` | `["URL"]` | 导航到 URL |
| `back` | `[]` | 后退 |
| `forward` | `[]` | 前进 |
| `reload` | `[]` | 重新加载页面 |

#### 读取内容
| 命令 | 参数 | 描述 |
|---------|------|-------------|
| `snapshot` | `["-i"]` | 带有 @ref 标签的交互式快照(最有用) |
| `text` | `[]` | 完整页面文本 |
| `html` | `["selector?"]` | 元素或完整页面的 HTML |
| `links` | `[]` | 页面上的所有链接 |
| `screenshot` | `["/tmp/s.png"]` | 截图 |
| `url` | `[]` | 当前 URL |

#### 交互
| 命令 | 参数 | 描述 |
|---------|------|-------------|
| `click` | `["@e3"]` | 点击元素(使用快照中的 @ref) |
| `fill` | `["@e5", "text"]` | 填写表单字段 |
| `select` | `["@e7", "option"]` | 选择下拉选项值 |
| `type` | `["text"]` | 输入文本(键盘) |
| `press` | `["Enter"]` | 按键 |
| `scroll` | `["down"]` | 滚动页面 |

#### 标签页
| 命令 | 参数 | 描述 |
|---------|------|-------------|
| `newtab` | `["URL?"]` | 创建新标签页(写入前必需) |
| `tabs` | `[]` | 列出所有标签页 |
| `closetab` | `["id?"]` | 关闭标签页 |

## 快照 → @ref 模式

这是最强大的浏览模式。无需编写 CSS 选择器:

1. 运行 `snapshot -i` 获取带有标记元素的交互式快照
2. 快照返回如下文本:
   ```
   [Page Title]
   @e1 [link] "Home"
   @e2 [button] "Sign In"
   @e3 [input] "Search..."
   ```
3. 在命令中直接使用 `@e` 引用:`click @e2`、`fill @e3 "search query"`

这就是快照系统的工作方式,比猜测 CSS 选择器可靠得多。始终先执行 `snapshot -i`,然后使用引用。

## 范围

| 范围 | 允许的操作 |
|-------|---------------|
| `read` | snapshot、text、html、links、screenshot、url、tabs、console 等 |
| `write` | goto、click、fill、scroll、newtab、closetab 等 |
| `admin` | eval、js、cookies、storage、cookie-import、useragent 等 |
| `meta` | tab、diff、frame、responsive、watch |

默认令牌获得 `read` + `write`。Admin 需要在配对时使用 `--admin` 标志。

## 标签页隔离

每个代理拥有它创建的标签页。规则:
- **读取:**任何代理都可以读取任何标签页(snapshot、text、screenshot)
- **写入:**只有标签页所有者可以写入(click、fill、goto 等)
- **无主标签页:**预先存在的标签页仅限 root 写入
- **第一步:**在尝试交互之前始终先执行 `newtab`

## 错误代码

| 代码 | 含义 | 应对措施 |
|------|---------|------------|
| 401 | 令牌无效、过期或已撤销 | 请用户再次运行 /pair-agent |
| 403 | 命令不在范围内,或标签页不属于您 | 使用 newtab,或请求 --admin |
| 429 | 超过速率限制(>10 请求/秒) | 等待 Retry-After 头指定的时间 |

## 安全模型

- **物理端口分离。**本地监听器和隧道监听器是独立的 TCP 套接字。ngrok 仅转发隧道端口。隧道调用者完全无法访问引导端点(404,错误端口)。
- **隧道命令白名单。**通过隧道的 `/command` 仅接受 26 个浏览器驱动命令(goto、click、fill、snapshot、text、newtab、tabs、back、forward、reload、closetab 等)。服务器管理命令(tunnel、pair、token、useragent、js)在隧道上被拒绝。
- **根令牌被隧道阻止。**通过隧道监听器携带根令牌的请求返回 403 并提示配对。只有受限会话令牌可以通过隧道工作。
- **设置密钥**在 5 分钟后过期,只能使用一次。
- **会话令牌**在 24 小时后过期(可配置)。
- 根令牌永远不会出现在指令块或连接字符串中。
- **Admin 范围**(JS 执行、cookie 访问)默认被拒绝。
- 令牌可以立即撤销:`$B tunnel revoke agent-name`
- **SSE 身份验证**使用 30 分钟 HttpOnly SameSite=Strict cookie,仅限流范围(对 `/command` 永远无效)。
- **路径遍历防护**在 `/welcome` 上 — `GSTACK_SLUG` 必须匹配 `^[a-z0-9_-]+$` 否则回退到内置模板。
- **SSRF 防护**在 `goto`、`download` 和抓取路径上 — 根据 localhost/私有范围黑名单验证 URL 目标。
- **隧道表面拒绝日志。**隧道监听器上的每次拒绝(`path_not_on_tunnel`、`root_token_on_tunnel`、`missing_scoped_token`、`disallowed_command:*`)都会附加到 `~/.gstack/security/attempts.jsonl`,包含时间戳、源 IP、路径、方法。速率限制为 60 次写入/分钟。
- 所有代理活动都记录有归属(clientId)。

**已知非目标(跟踪为 #1136):**在 Windows 上,cookie-import-browser 路径使用 `--remote-debugging-port=<random>` 启动 Chrome。使用 App-Bound Encryption v20,同用户本地进程可以连接到该端口并窃取解密的 v20 cookies — 相对于直接读取 SQLite DB 的提权路径。修复方向是使用 `--remote-debugging-pipe` 而不是 TCP。

## 同机器快捷方式

如果两个代理在同一台机器上,跳过复制粘贴:

```bash
$B pair-agent --local openclaw    # 写入 ~/.openclaw/skills/gstack/browse-remote.json
$B pair-agent --local codex       # 写入 ~/.codex/skills/gstack/browse-remote.json
$B pair-agent --local cursor      # 写入 ~/.cursor/skills/gstack/browse-remote.json
```

无需隧道。直接使用 localhost。

## ngrok 隧道设置

对于不同机器上的远程代理:

1. 在 [ngrok.com](https://ngrok.com) 注册(免费套餐可用)
2. 从仪表板复制您的身份验证令牌
3. 保存它:`echo 'NGROK_AUTHTOKEN=your_token' > ~/.gstack/ngrok.env`
4. 可选择声明一个稳定域名:`echo 'NGROK_DOMAIN=your-name.ngrok-free.dev' >> ~/.gstack/ngrok.env`
5. 使用隧道启动:`BROWSE_TUNNEL=1 $B restart`
6. 运行 `$B pair-agent` — 它将自动使用隧道 URL