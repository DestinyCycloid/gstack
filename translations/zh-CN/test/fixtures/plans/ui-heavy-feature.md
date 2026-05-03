# 计划:用户仪表板页面

## 背景
我们将在 `/dashboard` 推出一个新的用户仪表板,显示最近的活动、通知面板和快捷操作按钮。用户登录后将进入此页面。

## UI 范围
- 在 `src/pages/` 新建 React 页面组件 `UserDashboard.tsx`
- 三个新的子组件:`ActivityFeed`、`NotificationsPanel`、`QuickActions`
- 使用 Tailwind CSS 进行布局,移动优先响应式设计(断点:sm/md/lg)
- 每个面板的空状态、加载骨架屏、错误状态
- 每个交互元素的悬停状态 + focus-visible 轮廓
- 通知面板上"全部标记为已读"的模态对话框
- 用于操作反馈的 Toast 通知系统

## 后端
- 新的 REST 端点 `GET /api/dashboard` 返回 `{ activity, notifications, quickActions }`
- 基于现有的 PostgreSQL 表;无需更改架构

## 不在范围内
- 深色模式(单独计划)
- 个性化/自定义(单独计划)