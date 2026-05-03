# 性能专家审查清单

范围:当 SCOPE_BACKEND=true 或 SCOPE_FRONTEND=true 时
输出:JSON 对象,每行一个发现。模式:
{"severity":"CRITICAL|INFORMATIONAL","confidence":N,"path":"file","line":N,"category":"performance","summary":"...","fix":"...","fingerprint":"path:line:performance","specialist":"performance"}
可选字段:line、fix、fingerprint、evidence、test_stub。
如果没有发现:输出 `NO FINDINGS`,不输出其他内容。

---

## 类别

### N+1 查询
- 在循环中遍历 ActiveRecord/ORM 关联而未进行预加载(.includes、joinedload、include)
- 迭代块(each、map、forEach)内部的数据库查询本可以批量处理
- 触发延迟加载关联的嵌套序列化器
- 按字段查询而非批处理的 GraphQL 解析器(检查是否使用 DataLoader)

### 缺失数据库索引
- 在没有索引的列上新增 WHERE 子句(检查迁移文件或模式)
- 在未建立索引的列上新增 ORDER BY
- 没有复合索引的复合查询(WHERE a AND b)
- 添加的外键列没有索引

### 算法复杂度
- O(n^2) 或更差的模式:集合的嵌套循环、Array.map 内部的 Array.find
- 可以使用哈希/映射/集合查找的重复线性搜索
- 循环中的字符串拼接(应使用 join 或 StringBuilder)
- 对大型集合多次排序或过滤,而一次即可完成

### 打包体积影响(前端)
- 已知体积较大的新生产依赖(moment.js、完整的 lodash、jquery)
- 桶式导入(import from 'library')而非深度导入(import from 'library/specific')
- 提交的大型静态资源(图片、字体)未经优化
- 路由级代码块缺少代码拆分

### 渲染性能(前端)
- 获取瀑布流:可以并行的 API 调用却按顺序执行(Promise.all)
- 不稳定引用导致的不必要重新渲染(在渲染中创建新对象/数组)
- 昂贵的计算缺少 React.memo、useMemo 或 useCallback
- 循环中读取后写入 DOM 属性导致的布局抖动
- 折叠下方的图片缺少 loading="lazy"

### 缺失分页
- 返回无界结果的列表端点(无 LIMIT,无分页参数)
- 没有 LIMIT 且随数据量增长的数据库查询
- 嵌入完整嵌套对象而非带扩展的 ID 的 API 响应

### 异步上下文中的阻塞
- 异步函数内部的同步 I/O(文件读取、子进程、HTTP 请求)
- 基于事件循环的处理程序内部的 time.sleep() / Thread.sleep()
- CPU 密集型计算阻塞主线程而未使用 worker 卸载