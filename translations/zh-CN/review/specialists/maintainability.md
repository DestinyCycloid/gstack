```json
{"severity":"INFORMATIONAL","confidence":95,"path":"README.md","line":1,"category":"maintainability","summary":"文档标题重复","fix":"删除重复的标题行","fingerprint":"README.md:1:maintainability","specialist":"maintainability"}
```

---

# 可维护性专家审查清单

范围:始终启用(每次审查)
输出:JSON 对象,每行一个发现。模式:
{"severity":"INFORMATIONAL","confidence":N,"path":"file","line":N,"category":"maintainability","summary":"...","fix":"...","fingerprint":"path:line:maintainability","specialist":"maintainability"}
可选字段:line、fix、fingerprint、evidence、test_stub。
如果没有发现:输出 `NO FINDINGS`,不输出其他内容。

---

## 类别

### 死代码和未使用的导入
- 在更改的文件中已赋值但从未读取的变量
- 已定义但从未调用的函数/方法(使用 Grep 在整个仓库中检查)
- 更改后不再被引用的导入/require
- 被注释掉的代码块(要么删除,要么解释为什么存在)

### 魔法数字和字符串耦合
- 逻辑中使用的裸数字字面量(阈值、限制、重试次数)— 应该是命名常量
- 在其他地方用作查询过滤器或条件的错误消息字符串
- 应该是配置的硬编码 URL、端口或主机名
- 跨多个文件重复的字面量值

### 过时的注释和文档字符串
- 在此差异中代码更改后描述旧行为的注释
- 引用已完成工作的 TODO/FIXME 注释
- 参数列表与当前函数签名不匹配的文档字符串
- 注释中不再与代码流程匹配的 ASCII 图表

### DRY 违规
- 在差异中多次出现的相似代码块(3 行以上)
- 使用共享辅助函数会更清晰的复制粘贴模式
- 跨测试文件重复的配置或设置逻辑
- 可以是查找表或映射的重复条件链

### 条件副作用
- 根据条件分支但在某个分支上忘记副作用的代码路径
- 声称某个操作已发生但该操作被有条件跳过的日志消息
- 状态转换中一个分支更新相关记录但另一个分支不更新
- 仅在正常路径上触发的事件发射,缺少错误/边缘路径

### 模块边界违规
- 深入另一个模块的内部实现(访问按约定私有的方法)
- 控制器/视图中应该通过服务/模型进行的直接数据库查询
- 应该通过接口通信的组件之间的紧耦合