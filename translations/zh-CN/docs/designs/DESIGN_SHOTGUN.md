# 设计：Design Shotgun — 浏览器到 Agent 的反馈循环

生成于 2026-03-27
分支：garrytan/agent-design-tools
状态：活文档 — 发现并修复 bug 时更新

## 此功能的作用

Design Shotgun 生成多个 AI 设计模型，在用户的真实浏览器中并排打开它们作为对比面板，并收集结构化反馈（选择最喜欢的、评价备选方案、留下备注、请求重新生成）。反馈流回编码 agent，agent 据此采取行动：要么继续使用批准的变体，要么生成新变体并重新加载面板。

用户无需离开浏览器标签页。agent 不会问多余的问题。面板就是反馈机制。

## 核心问题：两个必须对话的世界

```
  ┌─────────────────────┐          ┌──────────────────────┐
  │   用户的浏览器       │          │   编码 AGENT         │
  │   (真实的 Chrome)    │          │   (Claude Code /     │
  │                     │          │    Conductor)         │
  │  对比面板           │          │                      │
  │  带按钮：           │   ???    │  需要知道：          │
  │  - 提交             │ ──────── │  - 选择了什么        │
  │  - 重新生成         │          │  - 星级评分          │
  │  - 更多类似         │          │  - 评论              │
  │  - 重新混合         │          │  - 请求重新生成？    │
  └─────────────────────┘          └──────────────────────┘
```

"???" 是困难的部分。用户在 Chrome 中点击按钮。在终端中运行的 agent 需要知道这件事。这是两个完全独立的进程，没有共享内存，没有共享事件总线，没有 WebSocket 连接。

## 架构：链接的工作原理

```
  用户的浏览器                      $D serve (Bun HTTP)              AGENT
  ═══════════════                   ═══════════════════              ═════
       │                                   │                           │
       │  GET /                            │                           │
       │ ◄─────── 提供面板 HTML ──────────►│                           │
       │    (在 <head> 中注入              │                           │
       │     __GSTACK_SERVER_URL)          │                           │
       │                                   │                           │
       │  [用户评分、选择、评论]           │                           │
       │                                   │                           │
       │  POST /api/feedback               │                           │
       │ ─────── {preferred:"A",...} ─────►│                           │
       │                                   │                           │
       │  ◄── {received:true} ────────────│                           │
       │                                   │── 写入 feedback.json ────►│
       │  [输入被禁用，                    │   (或 feedback-pending    │
       │   显示"返回 agent"]               │    .json 用于重新生成)    │
       │                                   │                           │
       │                                   │                  [agent 每 5 秒
       │                                   │                   轮询一次，
       │                                   │                   读取文件]
```

### 三个文件

| 文件 | 写入时机 | 含义 | Agent 操作 |
|------|---------|------|-----------|
| `feedback.json` | 用户点击提交 | 最终选择，完成 | 读取它，继续 |
| `feedback-pending.json` | 用户点击重新生成/更多类似 | 想要新选项 | 读取它，删除它，生成新变体，重新加载面板 |
| `feedback.json` (第 2 轮+) | 用户在重新生成后点击提交 | 迭代后的最终选择 | 读取它，继续 |

### 状态机

```
  $D serve 启动
       │
       ▼
  ┌──────────┐
  │ SERVING  │◄──────────────────────────────────────┐
  │          │                                        │
  │ 面板正在 │  POST /api/feedback                    │
  │ 运行，   │  {regenerated: true}                   │
  │ 等待中   │──────────────────►┌──────────────┐     │
  │          │                   │ REGENERATING │     │
  │          │                   │              │     │
  └────┬─────┘                   │ Agent 有     │     │
       │                         │ 10 分钟来    │     │
       │  POST /api/feedback     │ POST 新的    │     │
       │  {regenerated: false}   │ 面板 HTML    │     │
       │                         └──────┬───────┘     │
       ▼                                │             │
  ┌──────────┐                POST /api/reload        │
  │  DONE    │                {html: "/new/board"}    │
  │          │                          │             │
  │ exit 0   │                          ▼             │
  └──────────┘                   ┌──────────────┐     │
                                 │  RELOADING   │─────┘
                                 │              │
                                 │ 面板自动     │
                                 │ 刷新         │
                                 │ (同一标签页) │
                                 └──────────────┘
```

### 端口发现

agent 在后台运行 `$D serve` 并从 stderr 读取端口：

```
SERVE_STARTED: port=54321 html=/path/to/board.html
SERVE_BROWSER_OPENED: url=http://127.0.0.1:54321
```

agent 从 stderr 解析 `port=XXXXX`。稍后当用户请求重新生成时，需要此端口来 POST `/api/reload`。如果 agent 丢失端口号，它就无法重新加载面板。

### 为什么是 127.0.0.1，而不是 localhost

`localhost` 在某些系统上可能解析为 IPv6 `::1`，而 Bun.serve() 仅监听 IPv4。更重要的是，`localhost` 会发送开发者一直在处理的每个域的所有开发 cookie。在有许多活动会话的机器上，这会超过 Bun 的默认头大小限制（HTTP 431 错误）。`127.0.0.1` 避免了这两个问题。

## 每个边缘情况和陷阱

### 1. 僵尸表单问题

**问题：** 用户提交反馈，POST 成功，服务器退出。但 HTML 页面仍在 Chrome 中打开。它看起来是交互式的。用户可能会编辑他们的反馈并再次点击提交。什么都不会发生，因为服务器已经消失了。

**修复：** 成功 POST 后，面板 JS：
- 禁用所有输入（按钮、单选按钮、文本区域、星级评分）
- 完全隐藏重新生成栏
- 将提交按钮替换为："反馈已收到！返回您的编码 agent。"
- 显示："想要进行更多更改？再次运行 `/design-shotgun`。"
- 页面变成已提交内容的只读记录

**实现位置：** `compare.ts:showPostSubmitState()`（第 484 行）

### 2. 死服务器问题

**问题：** 服务器超时（默认 10 分钟）或在用户仍打开面板时崩溃。用户点击提交。fetch() 静默失败。

**修复：** `postFeedback()` 函数有一个 `.catch()` 处理程序。网络故障时：
- 显示红色错误横幅："连接丢失"
- 在可复制的 `<pre>` 块中显示收集的反馈 JSON
- 用户可以将其复制粘贴到他们的编码 agent 中

**实现位置：** `compare.ts:showPostFailure()`（第 546 行）

### 3. 陈旧的重新生成加载器

**问题：** 用户点击重新生成。面板显示加载器并每 2 秒轮询一次 `/api/progress`。agent 崩溃或生成新变体花费太长时间。加载器永远旋转。

**修复：** 进度轮询有 5 分钟的硬超时（150 次轮询 x 2 秒间隔）。5 分钟后：
- 加载器替换为："出了点问题。"
- 显示："在您的编码 agent 中再次运行 `/design-shotgun`。"
- 轮询停止。页面变为信息性的。

**实现位置：** `compare.ts:startProgressPolling()`（第 511 行）

### 4. file:// URL 问题（原始 BUG）

**问题：** 技能模板最初使用 `$B goto file:///path/to/board.html`。但 `browse/src/url-validation.ts:71` 出于安全原因阻止 `file://` URL。回退 `open file://...` 打开用户的 macOS 浏览器，但 `$B eval` 轮询 Playwright 的无头浏览器（不同的进程，从未加载页面）。agent 永远轮询空 DOM。

**修复：** `$D serve` 通过 HTTP 提供服务。永远不要对面板使用 `file://`。`$D compare` 上的 `--serve` 标志将面板生成和 HTTP 服务合并为一个命令。

**证据：** 参见 `.context/attachments/image-v2.png` — 一个真实用户遇到了这个确切的 bug。agent 正确诊断：(1) `$B goto` 拒绝 `file://` URL，(2) 即使使用浏览守护进程也没有轮询循环。

### 5. 双击竞争

**问题：** 用户快速双击提交。两个 POST 请求到达服务器。第一个将状态设置为"done"并在 100 毫秒内安排 exit(0)。第二个在那 100 毫秒窗口期间到达。

**当前状态：** 未完全防护。`handleFeedback()` 函数在处理之前不检查状态是否已经是"done"。第二个 POST 会成功并写入第二个 `feedback.json`（无害，相同数据）。退出仍在 100 毫秒后触发。

**风险：** 低。面板在第一次成功的 POST 响应时禁用所有输入，因此第二次点击需要在约 1 毫秒内到达。而且两次写入都将包含相同的反馈数据。

**潜在修复：** 在 `handleFeedback()` 顶部添加 `if (state === 'done') return Response.json({error: 'already submitted'}, {status: 409})`。

### 6. 端口协调问题

**问题：** agent 在后台运行 `$D serve` 并从 stderr 解析 `port=54321`。agent 稍后需要此端口在重新生成期间 POST `/api/reload`。如果 agent 丢失上下文（对话压缩，上下文窗口填满），它可能不记得端口。

**当前状态：** 端口打印到 stderr 一次。agent 必须记住它。没有端口文件写入磁盘。

**潜在修复：** 在启动时在面板 HTML 旁边写入 `serve.pid` 或 `serve.port` 文件。agent 可以随时读取它：
```bash
cat "$_DESIGN_DIR/serve.port"  # → 54321
```

### 7. 反馈文件清理问题

**问题：** 重新生成轮次的 `feedback-pending.json` 留在磁盘上。如果 agent 在读取它之前崩溃，下一个 `$D serve` 会话会找到一个陈旧的文件。

**当前状态：** 解析器模板中的轮询循环说在读取后删除 `feedback-pending.json`。但这取决于 agent 完美地遵循指令。陈旧的文件可能会混淆新会话。

**潜在修复：** `$D serve` 可以在启动时检查并删除陈旧的反馈文件。或者：用时间戳命名文件（`feedback-pending-1711555200.json`）。

### 8. 顺序生成规则

**问题：** 底层 OpenAI GPT Image API 对并发图像生成请求进行速率限制。当 3 个 `$D generate` 调用并行运行时，1 个成功，2 个被中止。

**修复：** 技能模板必须明确说明："一次生成一个模型。不要并行化 `$D generate` 调用。" 这是提示级别的指令，而不是代码级别的锁。设计二进制文件不强制顺序执行。

**风险：** agent 被训练为并行化独立工作。没有明确的指令，它们会尝试同时运行 3 个生成。这浪费 API 调用和金钱。

### 9. AskUserQuestion 冗余

**问题：** 在用户通过面板提交反馈后（JSON 中包含首选变体、评分、评论），agent 再次问他们："您更喜欢哪个变体？" 这很烦人。面板的全部意义就是避免这种情况。

**修复：** 技能模板必须说明："不要使用 AskUserQuestion 询问用户的偏好。读取 `feedback.json`，它包含他们的选择。只使用 AskUserQuestion 确认您理解正确，而不是重新询问。"

### 10. CORS 问题

**问题：** 如果面板 HTML 引用外部资源（来自 CDN 的字体、图像），浏览器会发送带有 `Origin: http://127.0.0.1:PORT` 的请求。大多数 CDN 允许这样做，但有些可能会阻止它。

**当前状态：** 服务器不设置 CORS 头。面板 HTML 是自包含的（图像 base64 编码，样式内联），所以这在实践中不是问题。

**风险：** 对当前设计来说低。如果面板加载外部资源会很重要。

### 11. 大负载问题

**问题：** 对 `/api/feedback` 的 POST 主体没有大小限制。如果面板以某种方式发送多 MB 负载，`req.json()` 会将其全部解析到内存中。

**当前状态：** 实际上，反馈 JSON 约为 500 字节到约 2KB。风险是理论上的，而不是实际的。面板 JS 构造一个固定形状的 JSON 对象。

### 12. fs.writeFileSync 错误

**问题：** `serve.ts:138` 中的 `feedback.json` 写入使用 `fs.writeFileSync()`，没有 try/catch。如果磁盘已满或目录是只读的，这会抛出异常并使服务器崩溃。用户看到永远的加载器（服务器已死，但面板不知道）。

**风险：** 实际上低（面板 HTML 刚刚写入同一目录，证明它是可写的）。但带有 500 响应的 try/catch 会更干净。

## 完整流程（逐步）

### 快乐路径：用户第一次就选择

```
1. Agent 运行：$D compare --images "A.png,B.png,C.png" --output board.html --serve &
2. $D serve 在随机端口上启动 Bun.serve()（例如 54321）
3. $D serve 在用户的浏览器中打开 http://127.0.0.1:54321
4. $D serve 打印到 stderr：SERVE_STARTED: port=54321 html=/path/board.html
5. $D serve 写入注入了 __GSTACK_SERVER_URL 的面板 HTML
6. 用户看到并排显示 3 个变体的对比面板
7. 用户选择选项 B，评分 A: 3/5，B: 5/5，C: 2/5
8. 用户在整体反馈中写道"B 的间距更好，就用它"
9. 用户点击提交
10. 面板 JS POST 到 http://127.0.0.1:54321/api/feedback
    主体：{"preferred":"B","ratings":{"A":3,"B":5,"C":2},"overall":"B has better spacing","regenerated":false}
11. 服务器将 feedback.json 写入磁盘（在 board.html 旁边）
12. 服务器将反馈 JSON 打印到 stdout
13. 服务器响应 {received:true, action:"submitted"}
14. 面板禁用所有输入，显示"返回您的编码 agent"
15. 服务器在 100 毫秒后以代码 0 退出
16. Agent 的轮询循环找到 feedback.json
17. Agent 读取它，向用户总结，继续
```

### 重新生成路径：用户想要不同的选项

```
1-6.  与上述相同
7.  用户点击"完全不同"标签
8.  用户点击重新生成
9.  面板 JS POST 到 /api/feedback
    主体：{"regenerated":true,"regenerateAction":"different","preferred":"","ratings":{},...}
10. 服务器将 feedback-pending.json 写入磁盘
11. 服务器状态 → "regenerating"
12. 服务器响应 {received:true, action:"regenerate"}
13. 面板显示加载器："正在生成新设计..."
14. 面板开始每 2 秒轮询一次 GET /api/progress

    同时，在 agent 中：
15. Agent 的轮询循环找到 feedback-pending.json
16. Agent 读取它，删除它
17. Agent 运行：$D variants --brief "totally different direction" --count 3
    (一次一个，不并行)
18. Agent 运行：$D compare --images "new-A.png,new-B.png,new-C.png" --output board-v2.html
19. Agent POST：curl -X POST http://127.0.0.1:54321/api/reload -d '{"html":"/path/board-v2.html"}'
20. 服务器将 htmlContent 交换为新面板
21. 服务器状态 → "serving"（从 reloading）
22. 面板的下一次 /api/progress 轮询返回 {"status":"serving"}
23. 面板自动刷新：window.location.reload()
24. 用户看到带有 3 个新变体的新面板
25. 用户选择一个，点击提交 → 从步骤 10 开始的快乐路径
```

### "更多类似"路径

```
与重新生成相同，除了：
- regenerateAction 是 "more_like_B"（引用变体）
- Agent 使用 $D iterate --image B.png --brief "more like this, keep the spacing"
  而不是 $D variants
```

### 回退路径：$D serve 失败

```
1. Agent 尝试 $D compare --serve，失败（二进制文件丢失，端口错误等）
2. Agent 回退到：open file:///path/board.html
3. Agent 使用 AskUserQuestion："我已经打开了设计面板。您更喜欢哪个变体？
   有什么反馈吗？"
4. 用户以文本形式回应
5. Agent 使用文本反馈继续（没有结构化 JSON）
```

## 实现此功能的文件

| 文件 | 角色 |
|------|------|
| `design/src/serve.ts` | HTTP 服务器、状态机、文件写入、浏览器启动 |
| `design/src/compare.ts` | 面板 HTML 生成、评分/选择/重新生成的 JS、POST 逻辑、提交后生命周期 |
| `design/src/cli.ts` | CLI 入口点，连接 `serve` 和 `compare --serve` 命令 |
| `design/src/commands.ts` | 命令注册表，定义带有参数的 `serve` 和 `compare` |
| `scripts/resolvers/design.ts` | `generateDesignShotgunLoop()` — 输出轮询循环和重新加载指令的模板解析器 |
| `design-shotgun/SKILL.md.tmpl` | 编排完整流程的技能模板：上下文收集、变体生成、`{{DESIGN_SHOTGUN_LOOP}}`、反馈确认 |
| `design/test/serve.test.ts` | HTTP 端点和状态转换的单元测试 |
| `design/test/feedback-roundtrip.test.ts` | E2E 测试：浏览器点击 → JS fetch → HTTP POST → 磁盘上的文件 |
| `browse/test/compare-board.test.ts` | 对比面板 UI 的 DOM 级测试 |

## 可能仍然出错的地方

### 已知风险（按可能性排序）

1. **Agent 不遵循顺序生成规则** — 大多数 LLM 想要并行化。没有在二进制文件中强制执行，这是一个可以被忽略的提示级别指令。

2. **Agent 丢失端口号** — 上下文压缩丢弃 stderr 输出。agent 无法重新加载面板。缓解措施：将端口写入文件。

3. **陈旧的反馈文件** — 崩溃会话遗留的 `feedback-pending.json` 混淆下一次运行。缓解措施：启动时清理。

4. **fs.writeFileSync 崩溃** — 反馈文件写入没有 try/catch。如果磁盘已满，服务器静默死亡。用户看到无限加载器。

5. **进度轮询漂移** — `setInterval(fn, 2000)` 超过 5 分钟。实际上，JavaScript 计时器足够准确。但如果浏览器标签页在后台，Chrome 可能会将间隔限制为每分钟一次。

### 运行良好的事情

1. **双通道反馈** — 前台模式的 stdout，后台模式的文件。两者始终活动。agent 可以使用任何有效的。

2. **自包含 HTML** — 面板内联所有 CSS、JS 和 base64 编码的图像。没有外部依赖。离线工作。

3. **同标签页重新生成** — 用户停留在一个标签页中。面板通过 `/api/progress` 轮询 + `window.location.reload()` 自动刷新。没有标签页爆炸。

4. **优雅降级** — POST 失败显示可复制的 JSON。进度超时显示清晰的错误消息。没有静默失败。

5. **提交后生命周期** — 面板在提交后变为只读。没有僵尸表单。清晰的"下一步做什么"消息。

## 测试覆盖率

### 已测试的内容

| 流程 | 测试 | 文件 |
|------|------|------|
| 提交 → 磁盘上的 feedback.json | 浏览器点击 → 文件 | `feedback-roundtrip.test.ts` |
| 提交后 UI 锁定 | 输入禁用，显示成功 | `feedback-roundtrip.test.ts` |
| 重新生成 → feedback-pending.json | 标签 + 重新生成点击 → 文件 | `feedback-roundtrip.test.ts` |
| "更多类似" → 特定操作 | JSON 中的 more_like_B | `feedback-roundtrip.test.ts` |
| 重新生成后的加载器 | DOM 显示加载文本 | `feedback-roundtrip.test.ts` |
| 完整重新生成 → 重新加载 → 提交 | 2 轮往返 | `feedback-roundtrip.test.ts` |
| 服务器在随机端口上启动 | 端口 0 绑定 | `serve.test.ts` |
| 服务器 URL 的 HTML 注入 | __GSTACK_SERVER_URL 检查 | `serve.test.ts` |
| 无效 JSON 拒绝 | 400 响应 | `serve.test.ts` |
| HTML 文件验证 | 如果丢失则 exit 1 | `serve.test.ts` |
| 超时行为 | 超时后 exit 1 | `serve.test.ts` |
| 面板 DOM 结构 | 单选按钮、星星、标签 | `compare-board.test.ts` |

### 未测试的内容

| 差距 | 风险 | 优先级 |
|-----|------|--------|
| 双击提交竞争 | 低 — 输入在第一次响应时禁用 | P3 |
| 进度轮询超时（150 次迭代） | 中 — 在测试中等待 5 分钟很长 | P2 |
| 重新生成期间服务器崩溃 | 中 — 用户看到无限加载器 | P2 |
| POST 期间网络超时 | 低 — localhost 很快 | P3 |
| 后台 Chrome 标签页限制间隔 | 中 — 可能将 5 分钟超时延长到 30+ 分钟 | P2 |
| 大反馈负载 | 低 — 面板构造固定形状的 JSON | P3 |
| 并发会话（两个面板，一个服务器） | 低 — 每个 $D serve 获得自己的端口 | P3 |
| 先前会话的陈旧反馈文件 | 中 — 可能混淆新的轮询循环 | P2 |

## 潜在改进

### 短期（此分支）

1. **将端口写入文件** — `serve.ts` 在启动时将 `serve.port` 写入磁盘。agent 随时读取它。5 行。
2. **启动时清理陈旧文件** — `serve.ts` 在启动前删除 `feedback*.json`。3 行。
3. **防护双击** — 在 `handleFeedback()` 顶部检查 `state === 'done'`。2 行。
4. **try/catch 文件写入** — 在 try/catch 中包装 `fs.writeFileSync`，失败时返回 500。5 行。

### 中期（后续）

5. **WebSocket 而不是轮询** — 用 WebSocket 连接替换 `setInterval` + `GET /api/progress`。面板在新 HTML 准备好时获得即时通知。消除轮询漂移和后台标签页限制。serve.ts 中约 50 行 + compare.ts 中约 20 行。

6. **agent 的端口文件** — 在启动时将 `{"port": 54321, "pid": 12345, "html": "/path/board.html"}` 写入 `$_DESIGN_DIR/serve.json`。agent 读取此文件而不是解析 stderr。使系统对上下文丢失更加健壮。

7. **反馈模式验证** — 在写入之前根据 JSON 模式验证 POST 主体。尽早捕获格式错误的反馈，而不是在下游混淆 agent。

### 长期（设计方向）

8. **持久设计服务器** — 不是每个会话启动 `$D serve`，而是运行一个长期存在的设计守护进程（如浏览守护进程）。多