# TODOS.md 格式参考

`/ship`(步骤 5.5)和 `/plan-ceo-review`(TODOS.md 更新部分)共享的规范 TODOS.md 格式参考,以确保 TODO 项结构的一致性。

---

## 文件结构

```markdown
# TODOS

## <技能/组件>     ← 例如:## Browse、## Ship、## Review、## Infrastructure
<按 P0 优先排序的项目,然后是 P1、P2、P3、P4>

## Completed
<已完成的项目及完成标注>
```

**分区:** 按技能或组件组织(`## Browse`、`## Ship`、`## Review`、`## QA`、`## Retro`、`## Infrastructure`)。在每个分区内,按优先级排序项目(P0 在顶部)。

---

## TODO 项格式

每个项目是其分区下的一个 H3:

```markdown
### <标题>

**What:** 工作的单行描述。

**Why:** 它解决的具体问题或释放的价值。

**Context:** 足够的细节,使 3 个月后接手的人能够理解动机、当前状态以及从哪里开始。

**Effort:** S / M / L / XL
**Priority:** P0 / P1 / P2 / P3 / P4
**Depends on:** <前置条件,或"None">
```

**必填字段:** What、Why、Context、Effort、Priority
**可选字段:** Depends on、Blocked by

---

## 优先级定义

- **P0** — 阻塞:必须在下一个版本之前完成
- **P1** — 关键:应该在本周期内完成
- **P2** — 重要:在 P0/P1 清理完成后执行
- **P3** — 锦上添花:在获得采用/使用数据后重新审视
- **P4** — 未来某天:好主意,但不紧急

---

## 已完成项格式

当项目完成时,将其移至 `## Completed` 分区,保留其原始内容并附加:

```markdown
**Completed:** vX.Y.Z (YYYY-MM-DD)
```