```json
{"severity":"INFORMATIONAL","confidence":95,"category":"testing","summary":"缺少负向路径测试 - 处理错误、拒绝或无效输入的新代码路径没有相应的测试","fix":"为错误处理分支添加测试用例,验证异常情况下的行为","fingerprint":"testing:negative-path","specialist":"testing"}
{"severity":"INFORMATIONAL","confidence":95,"category":"testing","summary":"缺少负向路径测试 - 守卫子句和提前返回未经测试","fix":"为守卫条件和提前返回路径添加测试用例","fingerprint":"testing:guard-clauses","specialist":"testing"}
{"severity":"CRITICAL","confidence":90,"category":"testing","summary":"缺少负向路径测试 - try/catch、rescue 或错误边界中的错误分支没有失败路径测试","fix":"为每个错误处理分支添加失败场景测试","fingerprint":"testing:error-branches","specialist":"testing"}
{"severity":"CRITICAL","confidence":95,"category":"testing","summary":"缺少负向路径测试 - 权限/认证检查在代码中断言但从未测试"拒绝"情况","fix":"添加测试用例验证权限被拒绝时的行为","fingerprint":"testing:auth-denied","specialist":"testing"}
{"severity":"INFORMATIONAL","confidence":90,"category":"testing","summary":"缺少边界情况覆盖 - 边界值:零、负数、最大整数、空字符串、空数组、nil/null/undefined","fix":"为边界值和极端情况添加测试用例","fingerprint":"testing:boundary-values","specialist":"testing"}
{"severity":"INFORMATIONAL","confidence":85,"category":"testing","summary":"缺少边界情况覆盖 - 单元素集合(循环中的差一错误)","fix":"添加单元素和空集合的测试用例","fingerprint":"testing:single-element","specialist":"testing"}
{"severity":"INFORMATIONAL","confidence":85,"category":"testing","summary":"缺少边界情况覆盖 - 面向用户输入中的 Unicode 和特殊字符","fix":"添加包含特殊字符和 Unicode 的测试用例","fingerprint":"testing:unicode","specialist":"testing"}
{"severity":"INFORMATIONAL","confidence":80,"category":"testing","summary":"缺少边界情况覆盖 - 没有竞态条件测试的并发访问模式","fix":"添加并发场景和竞态条件的测试","fingerprint":"testing:concurrency","specialist":"testing"}
{"severity":"CRITICAL","confidence":95,"category":"testing","summary":"测试隔离违规 - 测试共享可变状态(类变量、全局单例、未清理的数据库记录)","fix":"确保每个测试独立运行,使用 setup/teardown 清理共享状态","fingerprint":"testing:shared-state","specialist":"testing"}
{"severity":"CRITICAL","confidence":90,"category":"testing","summary":"测试隔离违规 - 顺序依赖的测试(按顺序通过,随机化时失败)","fix":"重构测试使其可以以任意顺序运行","fingerprint":"testing:order-dependent","specialist":"testing"}
{"severity":"INFORMATIONAL","confidence":85,"category":"testing","summary":"测试隔离违规 - 测试依赖系统时钟、时区或区域设置","fix":"使用时间模拟或依赖注入来控制时间相关的测试","fingerprint":"testing:time-dependent","specialist":"testing"}
{"severity":"CRITICAL","confidence":95,"category":"testing","summary":"测试隔离违规 - 测试进行真实网络调用而不是使用 stubs/mocks","fix":"使用 mock 或 stub 替换真实的网络调用","fingerprint":"testing:real-network","specialist":"testing"}
{"severity":"CRITICAL","confidence":90,"category":"testing","summary":"不稳定测试模式 - 依赖时间的断言(sleep、setTimeout、waitFor 使用紧凑超时)","fix":"使用确定性等待或增加超时容差","fingerprint":"testing:timing-assertions","specialist":"testing"}
{"severity":"INFORMATIONAL","confidence":85,"category":"testing","summary":"不稳定测试模式 - 对无序结果的顺序断言(哈希键、Set 迭代、异步解析顺序)","fix":"对无序结果使用顺序无关的断言","fingerprint":"testing:unordered-results","specialist":"testing"}
{"severity":"CRITICAL","confidence":90,"category":"testing","summary":"不稳定测试模式 - 测试依赖外部服务(API、数据库)而没有后备方案","fix":"使用 mock 服务或本地测试替身","fingerprint":"testing:external-services","specialist":"testing"}
{"severity":"INFORMATIONAL","confidence":85,"category":"testing","summary":"不稳定测试模式 - 随机化测试数据没有种子控制","fix":"为随机数据生成器设置固定种子","fingerprint":"testing:random-data","specialist":"testing"}
{"severity":"CRITICAL","confidence":95,"category":"testing","summary":"缺少安全强制测试 - 控制器中的认证/授权检查没有"未授权"情况的测试","fix":"添加测试验证未授权访问被正确拒绝","fingerprint":"testing:auth-enforcement","specialist":"testing"}
{"severity":"CRITICAL","confidence":90,"category":"testing","summary":"缺少安全强制测试 - 限流逻辑没有测试证明它实际阻止了请求","fix":"添加测试验证限流机制正常工作","fingerprint":"testing:rate-limiting","specialist":"testing"}
{"severity":"CRITICAL","confidence":95,"category":"testing","summary":"缺少安全强制测试 - 输入清理没有恶意输入的测试","fix":"添加测试用例验证恶意输入被正确处理","fingerprint":"testing:input-sanitization","specialist":"testing"}
{"severity":"CRITICAL","confidence":90,"category":"testing","summary":"缺少安全强制测试 - CSRF/CORS 配置没有集成测试","fix":"添加集成测试验证 CSRF/CORS 保护正常工作","fingerprint":"testing:csrf-cors","specialist":"testing"}
{"severity":"INFORMATIONAL","confidence":95,"category":"testing","summary":"覆盖率缺口 - 新的公共方法/函数零测试覆盖率","fix":"为所有新的公共 API 添加测试","fingerprint":"testing:new-methods","specialist":"testing"}
{"severity":"INFORMATIONAL","confidence":90,"category":"testing","summary":"覆盖率缺口 - 已更改的方法,现有测试仅覆盖旧行为,不覆盖新分支","fix":"更新测试以覆盖新添加的代码路径","fingerprint":"testing:changed-methods","specialist":"testing"}
{"severity":"INFORMATIONAL","confidence":85,"category":"testing","summary":"覆盖率缺口 - 从多处调用的工具函数仅通过间接方式测试","fix":"为共享工具函数添加直接的单元测试","fingerprint":"testing:utility-functions","specialist":"testing"}
```