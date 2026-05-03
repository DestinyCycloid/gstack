# Conductor 会话流 API 提案

## 问题

当 Claude 通过 CDP 控制你的真实浏览器时(gstack `$B connect`),你需要查看两个窗口:**Conductor**(查看 Claude 的思考)和 **Chrome**(查看 Claude 的操作)。

gstack 的 Chrome 扩展侧边面板显示浏览活动——每个命令、结果和错误。但要实现*完整*的会话镜像(Claude 的思考、工具调用、代码编辑),侧边面板需要 Conductor 暴露对话流。

## 这将实现什么

在 gstack Chrome 扩展侧边面板中添加"会话"标签页,显示:
- Claude 的思考/内容(为性能考虑进行截断)
- 工具调用名称 + 图标(Edit、Bash、Read 等)
- 带成本估算的轮次边界
- 随着对话进展的实时更新

用户可以在一个地方看到所有内容——Claude 在浏览器中的操作 + Claude 在侧边面板中的思考——无需切换窗口。

## 提议的 API

### `GET http://127.0.0.1:{PORT}/workspace/{ID}/session/stream`

服务器发送事件端点,以 NDJSON 事件形式重新发送 Claude Code 的对话。

**事件类型**(复用 Claude Code 的 `--output-format stream-json` 格式):

```
event: assistant
data: {"type":"assistant","content":"Let me check that page...","truncated":true}

event: tool_use
data: {"type":"tool_use","name":"Bash","input":"$B snapshot","truncated_input":true}

event: tool_result
data: {"type":"tool_result","name":"Bash","output":"[snapshot output...]","truncated_output":true}

event: turn_complete
data: {"type":"turn_complete","input_tokens":1234,"output_tokens":567,"cost_usd":0.02}
```

**内容截断:** 流中的工具输入/输出限制为 500 字符。完整数据保留在 Conductor 的 UI 中。侧边面板是摘要视图,不是替代品。

### `GET http://127.0.0.1:{PORT}/api/workspaces`

发现端点,列出活动的工作空间。

```json
{
  "workspaces": [
    {
      "id": "abc123",
      "name": "gstack",
      "branch": "garrytan/chrome-extension-ctrl",
      "directory": "/Users/garry/gstack",
      "pid": 12345,
      "active": true
    }
  ]
}
```

Chrome 扩展通过将浏览服务器的 git 仓库(来自 `/health` 响应)与工作空间的目录或名称匹配来自动选择工作空间。

## 安全性

- **仅限本地主机。** 与 Claude Code 自身调试输出相同的信任模型。
- **无需认证。** 如果 Conductor 需要认证,在工作空间列表中包含一个 Bearer token,扩展在 SSE 请求时传递。
- **内容截断**是一项隐私功能——长代码输出、文件内容和敏感工具结果永远不会离开 Conductor 的完整 UI。

## gstack 需要构建的内容(扩展端)

侧边面板"会话"标签页中已有脚手架(当前显示占位符)。

当 Conductor 的 API 可用时:
1. 侧边面板通过端口探测或手动输入发现 Conductor
2. 获取 `/api/workspaces`,匹配到浏览服务器的仓库
3. 打开 `EventSource` 连接到 `/workspace/{id}/session/stream`
4. 渲染:助手消息、工具名称 + 图标、轮次边界、成本
5. 优雅降级:"连接 Conductor 以查看完整会话视图"

预计工作量:`sidepanel.js` 中约 200 行代码。

## Conductor 需要构建的内容(服务器端)

1. SSE 端点,按工作空间重新发送 Claude Code 的 stream-json
2. `/api/workspaces` 发现端点,包含活动工作空间列表
3. 内容截断(工具输入/输出 500 字符上限)

预计工作量:如果 Conductor 已经在内部捕获 Claude Code 流(用于自身 UI 渲染),约 100-200 行代码。

## 设计决策

| 决策 | 选择 | 理由 |
|----------|--------|-----------|
| 传输方式 | SSE(非 WebSocket) | 单向、自动重连、更简单 |
| 格式 | Claude 的 stream-json | Conductor 已经解析此格式;无需新架构 |
| 发现机制 | HTTP 端点(非文件) | Chrome 扩展无法读取文件系统 |
| 认证 | 无(本地主机) | 与浏览服务器、CDP 端口、Claude Code 相同 |
| 截断 | 500 字符 | 侧边面板宽度约 300px;长内容无用 |