---
title: Admin Web 产品与前端架构
date: 2026-08-13
tags:
  - ai-coding-remote
  - admin-platform
  - frontend
  - observability
aliases:
  - Admin Web D 方案
  - Diagnostics Console UI
status: accepted
related:
  - "[[Admin 与可观测平台架构]]"
  - "[[Admin 能力目录与导航策略]]"
  - "[[日志采集与诊断数据流]]"
  - "[[ADR-006 Admin Web 一次性重构]]"
---

# Admin Web 产品与前端架构

## 1. 决策摘要

Admin Web 采用 D 方案：以 Linear 式克制工作台作为全局外壳，以 Datadog Log Explorer 的 Facet、时间范围和高密度事件表作为 Events 核心，以 Sentry Issue Details 的事故叙事和证据时间线作为 Incidents 核心。

这不是对三个产品的视觉拼贴，而是按 CodexRemote 的任务重组：开发者先发现异常，再缩小时间和来源，检查单条事件，最后固定为可复查事故。

> [!important] 数据与部署结论
> 不新建数据库，不引入 Redis、Elasticsearch 或 ClickHouse。继续使用现有 PostgreSQL 和附件目录。页面查询能力通过现有表的查询 API、必要索引和服务端聚合实现；线上大规模日志仍按既定架构迁移到阿里云 SLS。

## 2. 使用者与核心任务

第一使用者是开发者和运维人员，后台运行在本地 Mac。优先级从高到低为：

1. 判断 iPhone、Relay、Mac Agent、Collector 是否健康。
2. 在一个明确时间窗口中找出异常事件。
3. 通过 `session_id`、`trace_id`、`turn_ref` 还原跨服务顺序。
4. 打开单条事件检查结构化字段和原始证据引用。
5. 将时间窗口冻结为 Incident，供人和 AI 复查。
6. 发起受控设备采集并管理诊断附件。

用户信息和用户行为是已经确认的产品方向，未实现阶段在导航中显示“规划中”，但不创建空页面。审计、设置及其他低确定性能力默认隐藏，只在 [[Admin 能力目录与导航策略]] 中保留重新评估条件。

## 3. 设计原则

- **工作优先**：首屏直接进入可用工作台，不放欢迎页、营销 Hero 或说明卡片。
- **受控密度**：列表适合扫描，详情适合阅读；不把每项数据包装成卡片。
- **状态可证**：环境、时间范围、数据源、实时状态、截断和延迟始终可见。
- **渐进展开**：默认展示决策所需字段，完整 JSON、关联和附件按需展开。
- **URL 即查询**：筛选、排序、时间范围和选中事件进入 URL，可刷新、收藏和分享。
- **人机同源**：页面和 AI API 读取相同查询结果、Incident Snapshot 和 provenance。
- **本地可靠**：字体、图标和运行资源随项目构建，不依赖互联网 CDN。
- **状态真实**：可用能力可以进入，规划中能力只表达方向，候选能力只留在文档；三者不能混用。

## 4. 信息架构

正式实现以下导航，不保留当前顶部四个 Tab：

```text
CodexRemote Admin
├── Observability
│   ├── Overview
│   ├── Events
│   └── Incidents
├── Collection & Evidence
│   ├── Captures
│   └── Artifacts
├── Product
│   ├── Users [规划中]
│   └── User Behavior [规划中]
└── System
    └── Services
```

`Users` 和 `User Behavior` 显示为不可点击的“规划中”，不注册路由、不请求数据、不进入空页面。`Sessions`、`Devices`、`Audit`、`Settings`、告警、团队、多租户和计费等候选能力完全隐藏，其价值、依赖和启用条件由 [[Admin 能力目录与导航策略]] 管理。

路由为页面事实来源：

| 页面 | 路由 | 职责 |
| --- | --- | --- |
| Overview | `/overview` | 系统摘要、异常趋势、积压和最近事故 |
| Events | `/events` | 日志探索、筛选、关联和事件详情 |
| Incidents | `/incidents`、`/incidents/:id` | 事故列表、证据窗口和调查时间线 |
| Captures | `/captures` | 设备发现、采集创建和任务状态 |
| Artifacts | `/artifacts` | 证据清单、校验、导入和下载 |
| Services | `/services` | Relay、Mac Agent、Collector、数据库状态 |

`/` 直接跳转 `/overview`。不保留基于 DOM class 切换的伪路由。

规划中项不出现在路由表中。导航视觉和无障碍行为遵循 [[Admin 能力目录与导航策略#3. 导航交互规则]]。

## 5. 页面规格

### 5.1 App Shell

- 左侧 232px 固定导航，支持压缩为 64px 图标栏。
- 导航显示 6 个可用入口以及“用户”“用户行为”两个规划中项；规划中项弱化且不可点击。
- 顶部上下文栏只放当前页面标题、环境、时间范围、全局搜索和实时状态。
- 主工作区使用完整剩余宽度，不放页面级悬浮大卡片。
- 导航项使用 Lucide 图标和短文本；陌生操作提供 tooltip。
- 1024px 宽度时压缩侧栏和次要列，事件表仍保留时间、级别、来源、事件四列。

### 5.2 Overview

- 顶部是单行运行摘要：在线服务、错误、警告、Collector backlog、摄入延迟。
- 中部使用一条 24 小时事件趋势带，按 `fault/error/warning/info` 分层。
- 下部左右分区：最近异常和服务矩阵；不是多层卡片墙。
- 点击指标必须进入带查询条件的 Events，而不是只显示静态数字。

### 5.3 Events Explorer

```text
┌ Facets ─────┬ Query + time range + live controls ───────────────┐
│ Source      │ Event volume histogram                             │
│ Profile     ├────────────────────────────────────────────────────┤
│ Level       │ Time | Level | Source | Event | Correlation       │
│ Category    │ ... dense, virtualized, cursor-paged rows ...      │
│ Event       ├───────────────────────────────────────────┬────────┤
│             │                                           │ Detail │
└─────────────┴───────────────────────────────────────────┴────────┘
```

- 左侧 Facet 显示当前查询范围内的计数，支持多选、排除和一键清除。
- 查询栏支持自由文本和结构化过滤器；第一阶段先提供可视化 Filter Builder，不自行发明完整查询语言。
- 默认时间范围为最近 30 分钟；支持绝对时间和 15m、30m、1h、6h、24h 快捷项。
- Histogram 由服务端聚合，点击柱区间收窄时间窗口。
- 事件列表使用游标分页与虚拟滚动；新事件到达时不破坏已选行和滚动位置。
- 实时模式只在用户位于最新时间窗口且未暂停时追加；查看历史或详情时显示“有新事件”，由用户决定刷新。
- 右侧详情为同一路由的选中状态，包含摘要、关联、字段、原始 JSON 和“建立事故”。
- 每个字段可执行“包含此值”“排除此值”“复制值”；这些操作修改 URL 查询。

### 5.4 Incidents

- 左侧为可筛选事故列表，按 `critical/warning/info`、`open/resolved` 和时间排序。
- 详情顶部显示标题、状态、严重性、证据窗口、来源分布和截断状态。
- 主体按调查顺序组织：摘要、锚点、跨服务时间线、附件、provenance、JSON Snapshot。
- 时间线使用统一事件行，不复制一套 Incident 专用事件样式。
- 创建事故使用可访问 Dialog；不再调用浏览器 `confirm()`。

### 5.5 Captures、Artifacts、Services

- Captures 将“设备”和“任务”放在同一工作区，创建动作只能选择白名单任务。
- Artifacts 是可排序表格，详情抽屉显示 Manifest、SHA-256、关联 Incident 和下载审计状态。
- Services 使用紧凑矩阵展示 profile 与组件，不在所有页面永久占用一整条 service pill 带。
- 全局导航只显示 Services 的聚合健康点；完整延迟和连接信息在 Services 页面查看。

## 6. 视觉系统

基调为“精密的本地工程工作台”：明亮中性色背景、深石墨文本、青绿色表示健康、琥珀表示警告、朱红表示错误，蓝色只用于可交互选中态。禁止紫色渐变、装饰光球、玻璃拟态和大面积单色。

| Token | 方向 |
| --- | --- |
| 字体 | 随包提供 IBM Plex Sans Variable；代码和 ID 使用 IBM Plex Mono |
| 圆角 | 控件 4px，Popover/Dialog 6px；不使用胶囊形文本容器 |
| 边框 | 1px 中性分隔，层级主要靠留白和色值而非阴影 |
| 行高 | 事件行 34-38px，普通表格 40px |
| 动效 | 120-180ms 状态过渡；尊重 `prefers-reduced-motion` |
| 图标 | `lucide-react`，16px 为主；图标按钮具备 label 和 tooltip |
| 数字 | 表格数字与时间使用 tabular numerals |

CSS 使用语义 Token，而不是页面内散落颜色：`--surface-canvas`、`--surface-raised`、`--text-primary`、`--border-subtle`、`--status-critical` 等。组件不得直接引用具体业务颜色。

## 7. 前端技术架构

采用一个可独立测试、最终由 Go embed 的前端应用：

```text
admin-platform/
├── web/
│   ├── src/
│   │   ├── app/           # Router、QueryClient、Shell
│   │   ├── features/      # overview/events/incidents/captures/artifacts/services
│   │   ├── components/    # 通用表格、筛选、Dialog、Drawer、状态组件
│   │   ├── api/           # 生成/手写的版本化 API client 和类型
│   │   ├── styles/        # tokens、reset、layout
│   │   └── test/
│   ├── package.json
│   ├── package-lock.json
│   ├── tsconfig.json
│   └── vite.config.ts
├── internal/server/web-dist/  # 构建产物，不手工编辑
└── internal/server/assets.go  # embed web-dist
```

依赖边界：

- React + TypeScript + Vite：应用与构建基础。
- React Router：真实页面路由和 URL 查询状态。
- TanStack Query：请求、缓存、失效和错误状态。
- TanStack Table + Virtual：高密度表格和虚拟滚动。
- Radix Primitives：Dialog、Popover、Tooltip 等无样式可访问状态组件。
- Lucide React：统一图标。
- ECharts：仅 Events histogram 和后续趋势图，懒加载，不承担页面布局。
- CSS Modules + 全局 Design Tokens：拥有自己的视觉实现，不引入整套通用 Admin UI 主题。

不采用 Ant Design Pro、Material UI 或整套 Dashboard 模板。它们会引入与本产品无关的样式、布局和兼容负担。

## 8. API 目标契约

所有页面 API 在切换版本中统一到 `/api/v1`，不保留旧别名：

| 能力 | 目标接口 |
| --- | --- |
| 全局摘要 | `GET /api/v1/overview` |
| 服务状态 | `GET /api/v1/services` |
| 事件查询 | `GET /api/v1/diagnostics/events` |
| 单事件 | `GET /api/v1/diagnostics/events/{id}` |
| Facets | `GET /api/v1/diagnostics/facets` |
| Histogram | `GET /api/v1/diagnostics/histogram` |
| Collector 摄入 | `POST /api/v1/ingest/events` |
| 实时失效通知 | `GET /api/v1/stream` |
| Incidents | `/api/v1/incidents...` |
| Devices/Captures | `/api/v1/devices`、`/api/v1/captures...` |
| Artifacts | `/api/v1/artifacts...` |

事件查询至少支持：`from`、`to`、多值 `source/profile/level/category/event`、`q`、`session_id`、`trace_id`、`turn_ref`、`cursor`、`limit`、`sort`。

所有列表响应包含 `items`、`next_cursor`、`truncated` 和 `meta`。`meta` 至少声明 `data_source`、`timezone`、`query_time_ms` 和实际时间窗口。前端不再兼容 `null` 数组、旧字段名或无版本响应。

## 9. 数据库影响

继续使用现有 PostgreSQL：

- `events`、`incidents`、`incident_events`、`incident_artifacts`、`captures`、`artifacts`、Collector 状态仍是权威数据。
- Histogram 使用 `date_bin` 或等价 PostgreSQL 聚合，不在浏览器对完整日志重新分桶。
- Facet 计数和时间查询先基于现有索引测量；只为真实慢查询添加组合或表达式索引。
- 第一阶段保存视图使用 URL，不增加 `saved_views` 表；等用户认证存在后再把个人视图持久化。
- 不复制 `events_v2`、不双写新旧表、不创建第二套数据库。

若列或索引在新契约中确认无消费者，使用单向迁移删除；不保留废弃列作为兼容影子。

## 10. 删除原则

新 Web 上线时必须删除，而不是隐藏：

- 当前 `index.html` 顶部 Tab 和单页 `.view` 切换结构。
- `app.js`、`incidents.js`、`ui.js` 的全局 DOM 查询、字符串模板和 `innerHTML` 渲染。
- 当前 `styles.css` 中 `.topbar`、`.tabs`、`.service-strip`、`.source-rail` 等旧视觉规则。
- 前端同时维护 `state`、DOM checked/class 和服务端查询的三份状态。
- 所有无版本旧 API 路由和返回结构。
- 旧 API 的前端容错分支、`null` 数组兼容和字段回退。
- 重复的事件行、状态徽标、空状态和时间格式实现。

开发分支中可以按步骤搭建新实现，但合并和部署时只有一套 Web、一路 API 和一份组件实现。

## 11. 可访问性与性能基线

- 所有功能可用键盘完成，焦点进入/退出 Dialog 和 Drawer 后位置正确恢复。
- 状态不只依赖颜色；文本、图标和 `aria-label` 同时表达。
- 正文和控件满足 WCAG AA 对比度；支持 200% 文本缩放。
- 1024、1280、1440 和 1728 宽度无重叠、截断失控或横向页面滚动。
- 事件列表每页最多 200 条，继续加载使用游标；DOM 行数由虚拟化限制。
- SSE 只触发相关 Query 失效，不整页重绘，不破坏选择、滚动和输入焦点。
- 图表单独懒加载；初始工作台不因图表库阻塞导航和事件表骨架。

## 12. 设计来源

- [Datadog Log Explorer](https://docs.datadoghq.com/logs/explorer/)：Facet、查询、时间图和日志详情工作流。
- [Sentry Issue Details](https://docs.sentry.io/product/issues/issue-details/)：问题摘要、影响、Breadcrumb/时间线和证据组织。
- [Linear Custom Views](https://linear.app/docs/custom-views)：克制导航、列表密度、Peek 式详情和 URL/视图心智模型。
- [Grafana Explore](https://grafana.com/docs/grafana/latest/visualizations/explore/)：时间窗口、日志量、原始数据和跨信号关联，用作工程能力参考而非视觉外壳。

正式实现应保持 CodexRemote 自己的语义、Token 和组件，不复制产品商标、文案或像素样式。
