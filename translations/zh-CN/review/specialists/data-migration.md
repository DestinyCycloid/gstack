# 数据迁移专家审查清单

范围:当 SCOPE_MIGRATIONS=true 时
输出:JSON 对象,每行一个发现。模式:
{"severity":"CRITICAL|INFORMATIONAL","confidence":N,"path":"file","line":N,"category":"data-migration","summary":"...","fix":"...","fingerprint":"path:line:data-migration","specialist":"data-migration"}
可选:line、fix、fingerprint、evidence、test_stub。
如果没有发现:输出 `NO FINDINGS`,不输出其他内容。

---

## 类别

### 可逆性
- 此迁移是否可以在不丢失数据的情况下回滚?
- 是否有相应的 down/rollback 迁移?
- 回滚是否真正撤销了更改,还是只是空操作?
- 回滚是否会破坏当前的应用程序代码?

### 数据丢失风险
- 删除仍包含数据的列(应先添加弃用期)
- 更改会截断数据的列类型(varchar(255) → varchar(50))
- 在未验证没有代码引用的情况下删除表
- 在未更新所有引用(ORM、原始 SQL、视图)的情况下重命名列
- 向包含现有 NULL 值的列添加 NOT NULL 约束(需要先回填)

### 锁定持续时间
- 在大表上执行 ALTER TABLE 而不使用 CONCURRENTLY(PostgreSQL)
- 在超过 10 万行的表上添加索引而不使用 CONCURRENTLY
- 多个 ALTER TABLE 语句本可以合并为一次锁获取
- 在高峰流量时段获取排他锁的模式更改

### 回填策略
- 没有 DEFAULT 值的新 NOT NULL 列(需要在约束之前回填)
- 需要批量填充计算默认值的新列
- 缺少用于现有记录的回填脚本或 rake 任务
- 一次性更新所有行而不是分批的回填(锁定表)

### 索引创建
- 在生产表上执行 CREATE INDEX 而不使用 CONCURRENTLY
- 重复索引(新索引覆盖与现有索引相同的列)
- 新外键列上缺少索引
- 部分索引在完整索引更有用的情况下使用(或反之)

### 多阶段安全性
- 必须与应用程序代码按特定顺序部署的迁移
- 破坏当前运行代码的模式更改(先部署代码,然后迁移)
- 假设部署边界的迁移(旧代码 + 新模式 = 崩溃)
- 缺少功能标志来处理滚动部署期间的新旧代码混合