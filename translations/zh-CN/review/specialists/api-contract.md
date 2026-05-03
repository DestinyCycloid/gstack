# API 契约专家审查清单

范围:当 SCOPE_API=true 时
输出:JSON 对象,每行一个发现。模式:
{"severity":"CRITICAL|INFORMATIONAL","confidence":N,"path":"file","line":N,"category":"api-contract","summary":"...","fix":"...","fingerprint":"path:line:api-contract","specialist":"api-contract"}
可选:line、fix、fingerprint、evidence、test_stub。
如果没有发现:输出 `NO FINDINGS`,不输出其他内容。

---

## 类别

### 破坏性变更
- 从响应体中删除字段(客户端可能依赖它们)
- 更改字段类型(string → number,object → array)
- 向现有端点添加新的必需参数
- 更改 HTTP 方法(GET → POST)或状态码(200 → 201)
- 重命名端点而未将旧路径保留为重定向/别名
- 更改身份验证要求(公开 → 需要身份验证)

### 版本控制策略
- 在没有版本升级的情况下进行破坏性变更(v1 → v2)
- 在同一 API 中混合使用多种版本控制策略(URL vs header vs query param)
- 弃用端点但没有提供终止时间表或迁移指南
- 版本特定逻辑分散在控制器中而不是集中管理

### 错误响应一致性
- 新端点返回与现有端点不同的错误格式
- 错误响应缺少标准字段(错误代码、消息、详细信息)
- HTTP 状态码与错误类型不匹配(错误返回 200,验证错误返回 500)
- 错误消息泄露内部实现细节(堆栈跟踪、SQL)

### 速率限制和分页
- 新端点缺少速率限制,而类似端点有速率限制
- 分页变更(offset → cursor)没有向后兼容性
- 更改页面大小或默认限制而没有文档说明
- 分页响应中缺少总数或下一页指示器

### 文档偏差
- OpenAPI/Swagger 规范未更新以匹配新端点或更改的参数
- README 或 API 文档在变更后仍描述旧行为
- 示例请求/响应不再有效
- 新端点或更改的参数缺少文档

### 向后兼容性
- 使用旧版本的客户端:它们会中断吗?
- 无法强制更新的移动应用:API 对它们仍然有效吗?
- Webhook 负载更改而未通知订阅者
- 使用新功能需要更改 SDK 或客户端库