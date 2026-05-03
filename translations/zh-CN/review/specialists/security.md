# 安全专家审查清单

范围:当 SCOPE_AUTH=true 或 (SCOPE_BACKEND=true 且 diff > 100 行)
输出:JSON 对象,每行一个发现。模式:
{"severity":"CRITICAL|INFORMATIONAL","confidence":N,"path":"file","line":N,"category":"security","summary":"...","fix":"...","fingerprint":"path:line:security","specialist":"security"}
可选:line、fix、fingerprint、evidence、test_stub。
如果没有发现:输出 `NO FINDINGS`,不输出其他内容。

---

此清单比主要的 CRITICAL 检查更深入。主代理已经检查了 SQL 注入、竞态条件、LLM 信任和枚举完整性。此专家专注于认证/授权模式、加密误用和攻击面扩展。

## 类别

### 信任边界的输入验证
- 在控制器/处理器层面接受用户输入而不进行验证
- 查询参数直接用于数据库查询或文件路径
- 请求体字段在没有类型检查或模式验证的情况下被接受
- 文件上传没有类型/大小/内容验证
- Webhook 负载在没有签名验证的情况下被处理

### 认证与授权绕过
- 端点缺少认证中间件(检查路由定义)
- 授权检查默认为"允许"而不是"拒绝"
- 角色提升路径(用户可以修改自己的角色/权限)
- 直接对象引用漏洞(用户 A 通过更改 ID 访问用户 B 的数据)
- 会话固定或会话劫持机会
- Token/API 密钥验证不检查过期时间

### 注入向量(SQL 之外)
- 通过带有用户控制参数的子进程调用进行命令注入
- 模板注入(Jinja2、ERB、Handlebars)使用用户输入
- 目录查询中的 LDAP 注入
- 通过用户控制的 URL 进行 SSRF(fetch、redirect、webhook 目标)
- 通过用户控制的文件路径进行路径遍历(../../etc/passwd)
- 通过 HTTP 头中用户控制的值进行头注入

### 加密误用
- 对安全敏感操作使用弱哈希算法(MD5、SHA1)
- 对 token 或密钥使用可预测的随机性(Math.random、rand())
- 对密钥、token 或摘要进行非恒定时间比较(==)
- 硬编码的加密密钥或 IV
- 密码哈希中缺少盐值

### 密钥暴露
- 源代码中的 API 密钥、token 或密码(即使在注释中)
- 密钥记录在应用程序日志或错误消息中
- URL 中的凭据(查询参数或 URL 中的基本认证)
- 返回给用户的错误响应中的敏感数据
- 在预期加密时以明文存储的 PII

### 通过转义漏洞的 XSS
- Rails:.html_safe、raw() 用于用户控制的数据
- React:dangerouslySetInnerHTML 用于用户内容
- Vue:v-html 用于用户内容
- Django:|safe、mark_safe() 用于用户输入
- 通用:innerHTML 赋值使用未清理的数据

### 反序列化
- 反序列化不受信任的数据(pickle、Marshal、YAML.load、可执行类型的 JSON.parse)
- 接受来自用户输入或外部 API 的序列化对象而不进行模式验证