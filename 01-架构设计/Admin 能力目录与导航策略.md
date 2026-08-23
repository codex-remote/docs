---
title: Admin 能力目录与导航策略
date: 2026-08-13
tags:
  - ai-coding-remote
  - admin-platform
  - product
  - navigation
aliases:
  - Admin Capability Catalog
  - 后台能力目录
status: accepted
related:
  - "[[Admin Web 产品与前端架构]]"
  - "[[Admin 与可观测平台架构]]"
  - "[[ADR-006 Admin Web 一次性重构]]"
---

# Admin 能力目录与导航策略

## 1. 目的

后台导航必须真实反映产品状态，同时保留未来扩展的可发现性。页面不能把架构设想伪装成已经可用的功能，也不能因为当前不展示就让未来能力从项目知识中消失。

本目录是 Admin 能力状态的产品事实来源。代码只实现 `available` 和明确要求展示的 `planned`；`candidate` 与 `retired` 只保留在文档中。

## 2. 四级状态

| 状态 | 中文语义 | 后台表现 | 路由/API | 适用条件 |
| --- | --- | --- | --- | --- |
| `available` | 可用 | 正常导航，可进入和操作 | 必须存在且通过 E2E | 已实现、数据真实、可完成核心任务 |
| `planned` | 规划中 | 导航显示名称和“规划中”，不可点击 | 不创建占位路由或空 API | 产品方向已明确，用户需要知道它会出现 |
| `candidate` | 候选 | 完全隐藏 | 不进入代码 | 价值或需求尚未确认，但值得保留设计线索 |
| `retired` | 已移除 | 完全隐藏 | 必须删除 | 已验证无价值、被合并或被新能力替代 |

> [!important] 文案决策
> 后台统一使用“规划中”，不使用“待开发”“敬请期待”或百分比进度。“规划中”表达产品方向，不暗示排期。详细依赖和启用条件只写在本文档，不把开发说明放进产品界面。

## 3. 导航交互规则

### Available

- 有真实页面、真实数据、加载/空/错误状态和权限语义。
- 点击后 URL 改变，刷新可恢复当前状态。
- 不能用静态示例、假数字或禁用控件冒充已交付能力。

### Planned

- 显示在对应导航分组末尾，颜色弱于可用项。
- 右侧使用紧凑文本标签“规划中”，不是醒目的营销徽章。
- 整行不可点击，不注册路由，不预取数据，不显示空页面。
- 可聚焦并由辅助技术读出“用户，规划中”；tooltip 只说明“此模块尚未开放”。
- 移动或收起侧栏时，隐藏文本标签，但 tooltip 和无障碍名称必须保留状态。

### Candidate / Retired

- 不出现在导航、命令菜单、搜索结果、快捷键、前端路由或 API client 中。
- 不创建隐藏 feature flag、空组件、占位目录或数据库表。
- 只能从本文档和关联 ADR 中发现。

## 4. 当前导航清单

### 可用能力

| ID | 导航分组 | 显示名称 | 路由 | 现有数据/服务依据 |
| --- | --- | --- | --- | --- |
| `overview` | 可观测性 | 总览 | `/overview` | Overview、Services、Incident 和事件聚合 |
| `events` | 可观测性 | 事件 | `/events` | PostgreSQL `events` 与 Diagnostics API |
| `incidents` | 可观测性 | 事故 | `/incidents` | Incident Snapshot v1 |
| `captures` | 采集与证据 | 采集 | `/captures` | CoreDevice 设备发现、Capture 租约与 Collector |
| `artifacts` | 采集与证据 | 证据库 | `/artifacts` | Artifact Manifest、本地文件与哈希 |
| `services` | 系统 | 服务 | `/services` | Relay、Mac Agent、Collector、PostgreSQL 状态 |

“可用”指新 D 方案必须使用已有真实能力完成页面，不表示当前旧页面已经达到目标视觉和交互规格。

### 规划中且显示

| ID | 导航分组 | 显示名称 | 后台标签 | 为什么显示 | 升级为可用的最低条件 |
| --- | --- | --- | --- | --- | --- |
| `users` | 产品 | 用户 | 规划中 | 已明确需要查看用户资料及其设备、会话和事故 | 用户身份模型、PostgreSQL Schema、查询 API、隐私权限和审计完成 |
| `behavior` | 产品 | 用户行为 | 规划中 | 已明确需要查看用户行为，并与技术事故关联 | 行为事件 Schema、采集链路、保留策略、查询 API 和隐私审查完成 |

规划中能力不显示假计数、假头像、示例趋势或预计上线时间。

## 5. 隐藏的扩展能力目录

### 已有邻近能力，暂不独立成页

| 能力 | 状态 | 隐藏原因 | 重新评估触发 |
| --- | --- | --- | --- |
| 会话 | `candidate` | 当前可通过 Users、Behavior 和事件关联呈现，独立导航可能重复 | 用户排障主要从会话开始，且列表量需要独立检索 |
| 设备 | `candidate` | 当前设备用途是发起采集，归入 Captures 更直接 | 出现设备注册、撤销、版本治理或多设备生命周期管理 |
| Collector 独立页 | `candidate` | 当前 heartbeat/backlog 可放在 Services | 多 Collector、分片、升级或任务调度需要独立运维 |
| 保存视图 | `candidate` | URL 已能保存和分享查询，不需要用户表 | 完成登录后，用户明确需要跨设备收藏和共享视图 |

### 安全与平台能力

| 能力 | 状态 | 隐藏原因 | 重新评估触发 |
| --- | --- | --- | --- |
| 审计 | `candidate` | 本地单用户阶段没有完整身份与 RBAC，空审计页无意义 | 引入登录、敏感资料查看、导出或远程 Admin |
| 设置 | `candidate` | 当前配置由环境变量和运维文件管理，通用 Settings 会成为杂物页 | 出现必须由管理员在线修改且可审计的配置 |
| 角色与权限 | `candidate` | 尚未进入多用户部署 | 引入第二个管理员角色或远程访问 |
| 团队/组织/多租户 | `candidate` | 当前产品是单用户、单环境 | 明确存在多个隔离组织及数据边界 |

### 可观测性扩展

| 能力 | 状态 | 隐藏原因 | 重新评估触发 |
| --- | --- | --- | --- |
| 告警规则 | `candidate` | 本地阶段没有通知目标和告警值班流程 | SLS 上线且形成明确告警响应流程 |
| Metrics/Traces/Profiles 独立页 | `candidate` | 当前主要证据是结构化事件与附件 | OpenTelemetry/SLS 数据稳定，并有跨信号排障需求 |
| 自定义 Dashboard Builder | `candidate` | 易产生低价值卡片和长期配置成本 | 多角色对不同稳定指标有持续需求 |
| 数据源管理 | `candidate` | 当前 Local PostgreSQL 是唯一查询源 | SLS 或第二日志源真正接入并需要切换权限 |
| 保留策略管理 | `candidate` | 当前策略由运维配置控制 | 多环境、法规或不同数据域需要在线审批 |

### 商业能力

| 能力 | 状态 | 隐藏原因 | 重新评估触发 |
| --- | --- | --- | --- |
| 计费与订阅 | `candidate` | 当前没有商业化模型 | 产品正式商业化并确定计量单位 |
| 用量配额 | `candidate` | 当前不存在租户或套餐限制 | 需要按用户/组织执行真实资源限制 |

## 6. 状态变更流程

任何能力变更状态都必须：

1. 在本文档修改状态、理由、依赖和日期。
2. 若改变服务或数据边界，新增或修订 ADR。
3. `candidate → planned`：必须由产品方向明确支持；只增加导航元数据，不创建页面。
4. `planned → available`：必须在同一个版本完成 Schema、API、页面、权限、文档和 E2E，再移除“规划中”。
5. `available → retired`：先确认数据保留/迁移，再删除导航、路由、API、组件和测试，不能只隐藏 UI。
6. 同步 Admin README、Changelog、自动化验收和相关 ADR。

## 7. 实现约束

前端只维护一个静态、类型化的导航清单：

```ts
type VisibleCapability = {
  id: "overview" | "events" | "incidents" | "captures" | "artifacts" | "services" | "users" | "behavior"
  label: string
  status: "available" | "planned"
  route?: string
}
```

- `available` 必须有 `route`；`planned` 禁止有 `route`。
- `candidate` 和 `retired` 禁止进入该清单，因此不会把文档蓝图打包进产品代码。
- 导航清单是显示状态的唯一代码来源，不能在组件中散落 `disabled` 判断。
- 不从后端下发未实现功能列表，不为规划中能力建数据库表或远程 feature flag。

## 8. 验收

- 导航准确显示 6 个可用项和 2 个“规划中”项。
- “用户”“用户行为”不可点击、无路由、无网络请求、无空页面。
- Candidate 列表中的名称不出现在页面 DOM、命令菜单和前端 bundle 文案中。
- 键盘和 VoiceOver 能区分可用导航与规划中能力。
- 将测试能力从 `planned` 改为 `available` 时，类型检查要求提供 route。
- 文档搜索能够发现所有隐藏候选及其重新评估条件。
