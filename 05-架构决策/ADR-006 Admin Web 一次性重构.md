---
title: ADR-006 Admin Web 一次性重构
date: 2026-08-13
tags:
  - ai-coding-remote
  - adr
  - admin-platform
  - frontend
aliases:
  - ADR-006
status: accepted
decision_date: 2026-08-13
related:
  - "[[Admin Web 产品与前端架构]]"
  - "[[Admin 能力目录与导航策略]]"
  - "[[ADR-003 MVP 演进兼容边界]]"
  - "[[ADR-005 Admin Platform 与 Relay 分离]]"
---

# ADR-006 Admin Web 一次性重构

## 状态

已接受。

## 背景

当前 Admin Web 用单个 HTML、CSS 和少量原生 JavaScript 完成本地闭环，证明了采集、查询、Incident、CoreDevice 和附件链路，但页面以顶部 Tab、全局 DOM 状态和字符串模板组织。它不适合继续扩展用户、行为、审计、复杂日志查询和跨服务调查，也已经出现重复状态、重复事件样式和视觉层级不足。

项目尚未正式发布，根据 [[ADR-003 MVP 演进兼容边界]] 不承担预发布前端和 Admin API 的兼容义务。

## 决策

- 采用 [[Admin Web 产品与前端架构]] 定义的 D 方案。
- 使用 React、TypeScript 和 Vite 重建 Admin Web；Go 继续嵌入并交付唯一构建产物。
- 使用 React Router、TanStack Query/Table/Virtual、Radix Primitives、Lucide 和按需 ECharts。
- 继续使用现有 PostgreSQL，不创建第二数据库，不为重构引入 Redis。
- 页面状态以路由和 URL 查询为事实来源，服务端返回版本化查询元数据。
- 能力状态以 [[Admin 能力目录与导航策略]] 为事实来源：明确规划可显示为不可点击项，候选与已移除能力不进入产品代码。
- Admin API 一次性统一到 `/api/v1`；Collector 和 Web 同版本切换。
- 新实现通过 E2E 后删除旧静态页面、旧 API、旧返回结构、重复组件和兼容分支。
- 不提供 feature flag、legacy 模式、旧页面入口或运行期双轨。

## 为什么不继续扩展原生 JavaScript

当前规模下原生 JavaScript 足够，但未来需要真实路由、复杂 URL 查询、分页缓存、虚拟列表、可访问 Dialog/Popover、跨页面共享组件和稳定测试。继续以 `innerHTML` 和手写全局状态扩展会让每个新模块重复处理生命周期、焦点、竞态和清理。

## 为什么不采用通用 Admin 模板

Ant Design Pro、Material Dashboard 等模板以 CRUD 表单和卡片总览为中心。CodexRemote 的首要任务是高密度日志探索和事故证据，不应由模板导航、Card、Table 和主题结构反向决定产品。

采用无样式 primitives 和自有 Token 可以复用可访问状态机制，同时保留领域化信息密度与视觉控制。

## 为什么不做兼容

兼容会同时保留：

- 两套路由和响应 Schema；
- 两套页面状态与测试；
- Collector 新旧摄入路径；
- 旧 CSS/DOM 和新组件；
- 不再需要的容错分支。

这些代码不会提高本地未发布系统的用户价值，只会掩盖实际消费者并增加维护成本。切换在同一开发版本中完成，失败时回退整个构建，而不是让新旧实现长期共存。

## 数据处理

不兼容不等于无理由删除证据。现有 PostgreSQL 事件、Incident、Capture 和 Artifact 是有效数据，将通过前向迁移继续使用。只有被新 Schema 明确认定为冗余且无消费者的列、索引或记录类型才删除，并在迁移测试中验证。

禁止为了页面重构创建镜像表、双写表或 `v2` 数据库。

## 结果

正面结果：

- 页面结构可以扩展到用户、行为和审计，而不污染 Observability 工作流。
- 查询、选择和实时状态有唯一来源。
- Events、Incidents 和 Collection 共享稳定组件与视觉语言。
- 清除原型代码，降低长期前端和 API 维护面。

代价：

- 增加 Node/npm 构建链和前端依赖更新责任。
- Go API 与 Collector 必须在同一版本原子升级。
- 重构完成前开发分支不可作为稳定本地后台部署。

## 验证条件

- 新 Web 是 Admin Server 唯一静态资源入口。
- 仓库中不存在旧无版本 API 和旧页面实现。
- Simulator 日志采集、查询、详情和 Incident 重启恢复闭环通过。
- 1024-1728px 浏览器交互、键盘与可访问性验证通过。
- PostgreSQL 没有新旧双写，Collector spool/checkpoint 语义不变。
