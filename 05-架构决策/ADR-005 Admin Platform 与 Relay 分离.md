---
title: ADR-005 Admin Platform 与 Relay 分离
date: 2026-08-13
tags:
  - ai-coding-remote
  - adr
  - admin-platform
  - observability
aliases:
  - ADR-005
status: accepted
decision_date: 2026-08-13
amended_date: 2026-08-13
related:
  - "[[系统总体架构]]"
  - "[[Admin 与可观测平台架构]]"
  - "[[日志采集与诊断数据流]]"
  - "[[ADR-004 三仓库独立开发]]"
---

# ADR-005 Admin Platform 与 Relay 分离

## 状态

已接受。

## 背景

系统需要统一查看 iPhone App、Relay Server 和 Mac Agent 的日志，持久化 MetricKit、崩溃日志和 sysdiagnose，并允许从 Mac 管理页面发起 CoreDevice 采集。后续同一后台还将扩展用户信息、用户行为、设备、事故和管理审计。

Relay 当前承担低延迟 WebSocket 通信。把日志摄入、复杂查询、用户管理和设备采集加入 Relay，会混合不同权限、资源消耗和故障模式，并让后台查询或大附件操作影响实时通信。

## 决策

新增第四个独立源码项目 `admin-platform`：

- Admin Server 提供 Admin Web、Admin API 和 Diagnostics API。
- Admin Web 与 Admin Server 同一服务交付，内部模块化。
- Diagnostics Collector 与 Admin Server 同仓但作为独立进程运行。
- Relay 保持实时通信职责，不保存原始日志、不查询 SLS、不执行 CoreDevice。
- Collector 通过 API 提交事件和回报任务，不直接写数据库。
- 第一阶段本地开发即使用 PostgreSQL 14+ 和本地附件目录。
- Collector 使用本地原子 checkpoint 与有界 spool，不直接写 PostgreSQL。
- 线上模式继续使用 PostgreSQL、阿里云 SLS 和对象存储。
- 第一阶段不引入 Redis；新增 Redis 必须由真实多实例协调或缓存需求支持。

## iPhone 决策

iPhone 开发环境与线上环境使用同一套本地日志、持久队列、批量上传、失败重试、采样和 MetricKit 逻辑。开发环境仅额外允许 Mac Collector 通过 CoreDevice App Data Container 主动拉取诊断文件。

线上 iPhone 通过 SLS iOS SDK 上传，并使用 STS 临时凭据；不内置长期 AccessKey。OOM/Jetsam 不依赖终止回调，而由强杀前落盘、下次启动补传、MetricKit、系统崩溃日志和 sysdiagnose 共同定位。

## 数据域决策

用户资料、用户行为、技术日志、管理审计和诊断附件分开建模、授权和保留：

- 用户资料：PostgreSQL 权威关系数据。
- 用户行为：稳定业务事件，线上进入 SLS。
- 技术日志：结构化诊断事件，本地 JSONL、线上 SLS。
- 管理审计：不可变审计流和结构化索引。
- 诊断附件：对象或文件存储，数据库只保存 Manifest 和哈希。

## 替代方案

### 将 Diagnostics 加入 Relay

拒绝。会放大 Relay 故障域，并混合 WebSocket、后台查询、设备控制和用户权限。

### 独立 Diagnostics 服务，再另建 Admin 服务

当前不采用。第一阶段会产生重复的身份、权限、页面和部署成本。Diagnostics 作为 Admin 模块足够清晰，达到独立扩缩容需求后再拆分。

### Collector 直接写数据库

拒绝。会把采集进程与 PostgreSQL Schema 绑定，削弱幂等、权限和迁移边界。

### 所有日志写 PostgreSQL 或 MySQL

拒绝。关系数据库用于控制面与业务数据，SLS 用于线上大规模日志、行为和分析。

### 本地使用 SQLite，线上再迁移 PostgreSQL

不再采用。当前开发机已有可用 PostgreSQL，第一阶段直接使用同一数据库产品可以更早验证迁移、JSONB、索引、RBAC 数据和备份恢复，避免维护 SQLite/PostgreSQL 两套 SQL。Collector 的离线可靠性仍由本地 checkpoint/spool 保证，不依赖 Admin 数据库可用。

## 结果

正面结果：

- Relay 和诊断平台可以独立发布、扩缩容和故障隔离。
- 开发与线上统一 PostgreSQL Schema，线上日志数据面仍可平滑使用 SLS。
- Admin 可自然扩展用户、行为、设备和审计。
- CoreDevice 权限集中在最小化 Collector 中。

代价：

- 增加第四个仓库和一个 Collector 进程。
- 需要维护两套协议：实时通信协议与 Admin/诊断 Schema。
- PostgreSQL 与 SLS 承担不同数据域，需要明确查询适配、保留和验证。
- 需要独立处理采集游标、幂等、附件生命周期和 RBAC。

## 验证条件

- Admin 或 Collector 停止时，Relay/Mac Agent 核心链路继续工作。
- 文件轮转、Collector 重启和重复提交不会丢失或重复展示事件。
- iPhone 开发和线上使用同一上传状态机，开发环境 CoreDevice 只作为补充。
- 用户行为与技术日志可用共享关联 ID 串联，但权限、Schema 和保留互不混淆。
- 页面、JSON/NDJSON 导出和 AI 事故包能显示数据源、时区、截断、采样和脱敏信息。
