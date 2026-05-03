# 计划：团队速度仪表板

## 背景

我们正在为工程经理构建一个仪表板，用于跟踪团队代码速度——每位工程师的提交数、PR 周期时间、审查延迟、CI 通过率。数据已经存在于 GitHub 中；我们只是将其聚合以供经理单一视图查看。

## 变更

1. 在 `src/dashboard/` 中新建 React 组件 `TeamVelocityDashboard`
2. REST API 端点 `GET /api/team/velocity?days=30` 返回聚合指标
3. 后台任务每 15 分钟从 GitHub 拉取数据到 Postgres
4. 简单的筛选 UI：团队、日期范围、指标

## 架构

- 前端：React + shadcn/ui
- 后端：Express + PostgreSQL
- 数据源：GitHub REST API（缓存 15 分钟）

## 待解决问题

- 我们是否应该支持每个团队多个仓库？
- 我们是显示单个工程师的姓名还是仅显示聚合数据？