# Chrome vs Chromium:为什么我们使用 Playwright 捆绑的 Chromium

## 最初的设想

当我们构建 `$B connect` 时,计划是连接到用户的**真实 Chrome 浏览器**——包含他们的 cookies、会话、扩展和打开的标签页。不再需要导入 cookie。设计要求:

1. `chromium.connectOverCDP(wsUrl)` 通过 CDP 连接到运行中的 Chrome
2. 优雅地退出 Chrome,使用 `--remote-debugging-port=9222` 重新启动
3. 访问用户的真实浏览上下文

这就是为什么 `chrome-launcher.ts` 存在(361 行浏览器二进制发现、CDP 端口探测和运行时检测代码),以及为什么该方法被称为 `connectCDP()`。

## 实际发生的情况

当通过 Playwright 的 `channel: 'chrome'` 启动时,真实的 Chrome 会静默阻止 `--load-extension`。扩展无法加载。我们需要扩展来支持侧边栏(活动动态、refs、聊天)。

实现回退到使用 Playwright 捆绑的 Chromium 的 `chromium.launchPersistentContext()`——它可以通过 `--load-extension` 和 `--disable-extensions-except` 可靠地加载扩展。但命名保持不变:`connectCDP()`、`connectionMode: 'cdp'`、`BROWSE_CDP_URL`、`chrome-launcher.ts`。

最初的设想(访问用户的真实浏览器状态)从未实现。我们每次都启动一个全新的浏览器——功能上与 Playwright 的 Chromium 相同,但有 361 行死代码和误导性的命名。

## 发现问题(2026-03-22)

在一次 `/office-hours` 设计会议中,我们追踪架构并发现:

1. `connectCDP()` 不使用 CDP——它调用 `launchPersistentContext()`
2. `connectionMode: 'cdp'` 具有误导性——它只是"有头模式"
3. `chrome-launcher.ts` 是死代码——它唯一的导入在一个无法访问的 `attemptReconnect()` 方法中
4. `preExistingTabIds` 是为保护我们从未连接的真实 Chrome 标签页而设计的
5. `$B handoff`(无头 → 有头)使用了不同的 API(`launch()` + `newContext()`),无法加载扩展,创建了两种不同的"有头"体验

## 修复方案

### 重命名
- `connectCDP()` → `launchHeaded()`
- `connectionMode: 'cdp'` → `connectionMode: 'headed'`
- `BROWSE_CDP_URL` → `BROWSE_HEADED`

### 删除
- `chrome-launcher.ts`(361 行代码)
- `attemptReconnect()`(死方法)
- `preExistingTabIds`(死概念)
- `reconnecting` 字段(死状态)
- `cdp-connect.test.ts`(已删除代码的测试)

### 统一
- `$B handoff` 现在使用 `launchPersistentContext()` + 扩展加载(与 `$B connect` 相同)
- 一种有头模式,而不是两种
- Handoff 免费提供扩展 + 侧边栏

### 功能门控
- 侧边栏聊天在 `--chat` 标志后
- `$B connect`(默认):仅活动动态 + refs
- `$B connect --chat`:+ 实验性独立聊天代理

## 架构(修复后)

```
浏览器状态:
  HEADLESS(默认)←→ HEADED($B connect 或 $B handoff)
     Playwright            Playwright(相同引擎)
     launch()              launchPersistentContext()
     不可见                 可见 + 扩展 + 侧边栏

侧边栏(正交附加组件,仅有头模式):
  Activity 标签    — 始终开启,显示实时浏览命令
  Refs 标签        — 始终开启,显示 @ref 覆盖层
  Chat 标签        — 通过 --chat 选择加入,实验性独立代理

数据桥接(侧边栏 → 工作区):
  侧边栏写入 .context/sidebar-inbox/*.json
  工作区通过 $B inbox 读取
```

## 为什么不使用真实的 Chrome?

当 Playwright 启动真实 Chrome 时,它会阻止 `--load-extension`。这是 Chrome 的安全特性——通过命令行参数加载的扩展在基于 Chromium 的浏览器中受到限制,以防止恶意扩展注入。

Playwright 捆绑的 Chromium 没有这个限制,因为它是为测试和自动化设计的。`ignoreDefaultArgs` 选项让我们可以绕过 Playwright 自己的扩展阻止标志。

如果我们想要访问用户的真实 cookies/会话,路径是:
1. Cookie 导入(已通过 `$B cookie-import` 实现)
2. Conductor 会话注入(未来——侧边栏向工作区代理发送消息)

而不是重新连接到真实的 Chrome。