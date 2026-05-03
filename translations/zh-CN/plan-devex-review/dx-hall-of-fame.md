# DX 名人堂参考

仅阅读当前审查阶段的部分。不要加载整个文件。

## 第 1 阶段:入门

**黄金标准:**
- **Stripe**: 7 行代码完成信用卡收费。登录后文档会预填充您的测试 API 密钥。Stripe Shell 在文档页面内运行 CLI。无需本地安装。
- **Vercel**: `git push` = 在全球 CDN 上部署带 HTTPS 的实时站点。每个 PR 都获得预览 URL。一个 CLI 命令:`vercel`。
- **Clerk**: `<SignIn />`、`<SignUp />`、`<UserButton />`。3 个 JSX 组件,开箱即用的邮箱、社交、MFA 认证。
- **Supabase**: 创建 Postgres 表,自动生成 REST API + 实时功能 + 自文档化文档。
- **Firebase**: `onSnapshot()`。3 行代码实现所有客户端的实时同步,内置离线持久化。
- **Twilio**: 控制台中的虚拟电话。无需购买号码即可发送/接收短信,无需信用卡。结果:激活率提升 62%。

**反模式:**
- 在提供任何价值之前要求邮箱验证(破坏流程)
- 沙盒环境前要求信用卡
- "选择你自己的冒险"式的多路径(决策疲劳;一条黄金路径获胜)
- API 密钥隐藏在设置中(Stripe 将它们预填充到代码示例中)
- 没有语言切换的静态代码示例
- 文档站点与控制台分离(上下文切换)

## 第 2 阶段:API/CLI/SDK 设计

**黄金标准:**
- **Stripe 前缀 ID**: `ch_` 表示收费,`cus_` 表示客户。自文档化。不可能传递错误的 ID 类型。
- **Stripe 可扩展对象**: 默认返回 ID 字符串。`expand[]` 内联获取完整对象。嵌套扩展最多 4 层。
- **Stripe 幂等性密钥**: 在变更操作上传递 `Idempotency-Key` 头。安全重试。无需担心"我是否重复收费?"。
- **Stripe API 版本控制**: 首次调用将账户固定到当天的版本。通过 `Stripe-Version` 头按请求测试新版本。
- **GitHub CLI**: 自动检测终端 vs 管道。终端中人类可读,管道时制表符分隔。`gh pr <tab>` 显示所有 PR 操作。
- **SwiftUI 渐进式披露**: 从 `Button("Save") { save() }` 到完全自定义,每个级别使用相同 API。
- **htmx**: HTML 属性替代 JS。总共 14KB。`hx-get="/search" hx-trigger="keyup changed delay:300ms"`。零构建步骤。
- **shadcn/ui**: 将源代码复制到你的项目中。你拥有每一行。无依赖,无版本冲突。

**反模式:**
- 啰嗦的 API:一个用户可见操作需要 5 次调用
- 命名不一致:`/users`(复数)vs `/user/123`(单数)vs `/create-order`(URL 中的动词)
- 隐式失败:200 OK 但错误嵌套在响应体中
- 上帝端点:47 种参数组合,每个子集有不同行为
- 需要文档的 API:首次调用前需要 3 页文档 = 仪式感太强

## 第 3 阶段:错误消息与调试

**错误质量的三个层级:**

**第 1 层,Elm(对话式编译器):**
```
-- TYPE MISMATCH ---- src/Main.elm
I cannot do addition with String values like this one:
42|   "hello" + 1
     ^^^^^^^
Hint: To put strings together, use the (++) operator instead.
```
第一人称,完整句子,精确位置,建议修复,延伸阅读。

**第 2 层,Rust(注释源码):**
```
error[E0308]: mismatched types
 --> src/main.rs:4:20
help: consider borrowing here
  |
4 |     let name: &str = &get_name();
  |                       +
```
错误代码链接到教程。主要 + 次要标签。帮助部分显示确切编辑。

**第 3 层,Stripe API(结构化带 doc_url):**
```json
{"error":{"type":"invalid_request_error","code":"resource_missing","message":"No such customer: 'cus_nonexistent'","param":"customer","doc_url":"https://stripe.com/docs/error-codes/resource-missing"}}
```
五个字段,零歧义。

**公式:** 发生了什么 + 为什么 + 如何修复 + 在哪里了解更多 + 导致问题的实际值。

**反模式:** TypeScript 将"你是否想要?"埋在长错误链的底部。最可操作的信息应该首先出现。

## 第 4 阶段:文档与学习

**黄金标准:**
- **Stripe 文档**: 三栏布局(导航 / 内容 / 实时代码)。登录时注入 API 密钥。语言切换器在所有页面持久化。悬停高亮。Stripe Shell 用于浏览器内 API 调用。构建并开源了 Markdoc。功能在文档完成前不发布。文档贡献影响绩效评估。
- 52% 的开发者因缺乏文档而受阻(Postman 2023)
- 拥有世界级文档的公司采用率增加 2.5 倍
- "文档即产品":与功能一起发布,否则功能不发布

## 第 5 阶段:升级与迁移路径

**黄金标准:**
- **Next.js**: `npx @next/codemod upgrade major`。一个命令升级 Next.js、React、React DOM,运行所有相关 codemod。
- **AG Grid**: v31+ 的每个版本都包含 codemod。
- **Stripe API 版本控制**: 内部一个代码库。按账户固定版本。破坏性更改永远不会让你惊讶。
- **Martin Fowler 的管道模式**: 组合小型、可测试的转换,而不是一个单体 codemod。
- Maven Central 中 21.9% 的破坏性更改未记录(Ochoa 等,2021)

## 第 6 阶段:开发环境与工具

**黄金标准:**
- **Bun**: 比 npm install 快 100 倍,比 Node.js 运行时快 4 倍。速度就是 DX。
- 平均每天 87 次中断;每次恢复需要 25 分钟。开发者每天仅编码 2-4 小时。
- DXI 每提高 1 点 = 每个开发者每周节省 13 分钟。
- **GitHub Copilot**: 任务完成速度提高 55.8%。PR 时间从 9.6 天降至 2.4 天。

## 第 7 阶段:社区与生态系统

- 开发工具在购买前需要约 14 次曝光(Matt Biilmann,Netlify)。与季度 OKR 周期不兼容。
- 拥有强大开发者体验的团队性能倍增器为 4-5 倍(DevEx 框架)。

## 第 8 阶段:DX 测量

**三个学术框架:**
1. **SPACE**(微软研究院,2021):满意度、性能、活动、沟通、效率。至少测量 3 个维度。
2. **DevEx**(ACM Queue,2023):反馈循环、认知负荷、心流状态。结合感知 + 工作流数据。
3. **Fagerholm & Munch**(IEEE,2012):认知、情感、意动。心理学的"心智三部曲"。

## Claude Code 技能 DX 检查清单

在审查 Claude Code 技能、MCP 服务器或 AI 代理工具的计划时使用。

- [ ] **AskUserQuestion 设计**: 每次调用一个问题。重新建立上下文(项目、分支、任务)。浏览器切换用于视觉反馈。
- [ ] **状态存储**: 全局(~/.tool/)vs 按项目($SLUG/)vs 按会话。仅追加 JSONL 用于审计跟踪。
- [ ] **渐进式同意**: 带标记文件的一次性提示。永不重复询问。可逆。
- [ ] **自动升级**: 带缓存 + 延迟退避的版本检查。迁移脚本。内联提供。
- [ ] **技能组合**: benefits-from 链。审查链接。带部分跳过的内联调用。
- [ ] **错误恢复**: 从失败中恢复。保留部分结果。检查点安全。
- [ ] **会话连续性**: 时间线事件。压缩恢复。跨会话学习。
- [ ] **有界自主性**: 明确的操作限制。破坏性操作的强制升级。审计跟踪。

参考实现:gstack 的 design-shotgun 循环、自动升级流程、渐进式同意、分层存储。