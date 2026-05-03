# QA 报告：{APP_NAME}

| 字段 | 值 |
|-------|-------|
| **日期** | {DATE} |
| **URL** | {URL} |
| **分支** | {BRANCH} |
| **提交** | {COMMIT_SHA} ({COMMIT_DATE}) |
| **PR** | {PR_NUMBER} ({PR_URL}) 或 "—" |
| **级别** | Quick / Standard / Exhaustive |
| **范围** | {SCOPE 或 "完整应用"} |
| **持续时间** | {DURATION} |
| **访问页面数** | {COUNT} |
| **截图数** | {COUNT} |
| **框架** | {DETECTED 或 "未知"} |
| **索引** | [所有 QA 运行](./index.md) |

## 健康评分：{SCORE}/100

| 类别 | 评分 |
|----------|-------|
| 控制台 | {0-100} |
| 链接 | {0-100} |
| 视觉 | {0-100} |
| 功能 | {0-100} |
| 用户体验 | {0-100} |
| 性能 | {0-100} |
| 无障碍 | {0-100} |

## 需要修复的前 3 个问题

1. **{ISSUE-NNN}: {title}** — {one-line description}
2. **{ISSUE-NNN}: {title}** — {one-line description}
3. **{ISSUE-NNN}: {title}** — {one-line description}

## 控制台健康状况

| 错误 | 次数 | 首次出现 |
|-------|-------|------------|
| {error message} | {N} | {URL} |

## 摘要

| 严重程度 | 数量 |
|----------|-------|
| 严重 | 0 |
| 高 | 0 |
| 中 | 0 |
| 低 | 0 |
| **总计** | **0** |

## 问题

### ISSUE-001: {Short title}

| 字段 | 值 |
|-------|-------|
| **严重程度** | critical / high / medium / low |
| **类别** | visual / functional / ux / content / performance / console / accessibility |
| **URL** | {page URL} |

**描述：** {What is wrong, expected vs actual.}

**重现步骤：**

1. 导航到 {URL}
   ![Step 1](screenshots/issue-001-step-1.png)
2. {Action}
   ![Step 2](screenshots/issue-001-step-2.png)
3. **观察：** {what goes wrong}
   ![Result](screenshots/issue-001-result.png)

---

## 已应用的修复（如适用）

| 问题 | 修复状态 | 提交 | 更改的文件 |
|-------|-----------|--------|---------------|
| ISSUE-NNN | verified / best-effort / reverted / deferred | {SHA} | {files} |

### 修复前后对比

#### ISSUE-NNN: {title}
**修复前：** ![Before](screenshots/issue-NNN-before.png)
**修复后：** ![After](screenshots/issue-NNN-after.png)

---

## 回归测试

| 问题 | 测试文件 | 状态 | 描述 |
|-------|-----------|--------|-------------|
| ISSUE-NNN | path/to/test | committed / deferred / skipped | description |

### 延迟的测试

#### ISSUE-NNN: {title}
**前置条件：** {setup state that triggers the bug}
**操作：** {what the user does}
**预期：** {correct behavior}
**延迟原因：** {reason}

---

## 发布就绪状态

| 指标 | 值 |
|--------|-------|
| 健康评分 | {before} → {after} ({delta}) |
| 发现的问题 | N |
| 已应用的修复 | N (已验证: X, 尽力而为: Y, 已回滚: Z) |
| 已延迟 | N |

**PR 摘要：** "QA 发现 N 个问题，修复了 M 个，健康评分 X → Y。"

---

## 回归（如适用）

| 指标 | 基线 | 当前 | 差异 |
|--------|----------|---------|-------|
| 健康评分 | {N} | {N} | {+/-N} |
| 问题 | {N} | {N} | {+/-N} |

**自基线以来已修复：** {list}
**自基线以来新增：** {list}