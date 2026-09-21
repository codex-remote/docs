---
title: ADR-007 PostgreSQL 权威运行时与 Redis 实时加速
date: 2026-08-14
tags:
  - ai-coding-remote
  - adr
  - accepted-design
  - postgresql
  - redis
  - reliability
aliases:
  - ADR-007
status: accepted
implementation_status: runtime-v1-mobile-web-implemented
completed: false
decision_date: 2026-08-14
related:
  - "[[可靠多端会话与运行时控制面升级方案]]"
  - "[[可靠多端运行时分阶段开发计划]]"
  - "[[Runtime PostgreSQL 数据表规范]]"
  - "[[Runtime SQLite 与 Valkey 数据结构规范]]"
  - "[[Run Server HTTPS 与 SSE 接口规范]]"
  - "[[ADR-001 控制面与实时事件分离]]"
  - "[[ADR-003 MVP 演进兼容边界]]"
  - "[[ADR-005 Admin Platform 与 Relay 分离]]"
---

# ADR-007 PostgreSQL 权威运行时与 Redis 实时加速

## 状态

> [!success] 决策主链已落地
> 阶段 0-6 与 Mobile Web Runtime Auth 已实现并完成自动验收。公网 TLS/限流、多实例、容量与备份硬化，以及 iPhone 迁移仍待实现。

## 背景

当前 MVP 的 Relay 使用进程内连接状态，客户端与 Agent 事件依赖瞬时 WebSocket。它不能可靠支持 iPhone 与 Web 同时查看、Run 幂等提交、断线重放、多实例路由和进程重启恢复。

讨论中比较了 SQL-first 全量事件、Redis-first 全量事实、客户端 WebSocket，以及独立后台 Reconciler。最终需要同时满足：

- 客户端低延迟看到流式结果。
- Run 身份、生命周期和终态不能受 Redis TTL、裁剪或重启影响。
- Redis 中尚未持久化的结果有一个独立、可重放副本。
- Web 能自动发现由 iPhone 创建的新 Run。
- 不把周期性 Reconciler 作为首阶段正确性的前提。

## 决策

### 1. 公网运行时边界

新增 Run Server 运行时边界，提供 HTTP 命令、快照、查询、Session SSE 和 Run SSE，公网边缘升级 HTTPS。它可以与 WebSocket Relay 位于 `relay-server` 仓库，但不能并入 Admin Platform。

客户端不再通过 WebSocket 连接 Relay。Mac Agent 继续通过主动 WSS 连接 WebSocket Relay。

### 2. 命令与生命周期采用 PostgreSQL-first

Run Server 在一个事务中写入 Run、Session 事件和 Transactional Outbox，事务提交后才返回 `202 Accepted + run_id`。Dispatcher 从 PostgreSQL Outbox 领取并投递命令。

因此 Redis 不作为命令的唯一队列，也不参与“命令是否已接受”的正确性判定。

### 3. 实时结果采用 Redis-first，但不是 Redis-only

Mac Agent 在发送每个结果事件前，先写入本地 SQLite Outbox，并为每个 Run 分配单调 `agent_sequence`。Relay 将事件写入 Redis Result Stream 后返回 `received_ack`，当前集成式 Runtime Event Persister 再幂等写入 PostgreSQL并返回 `durable_ack`。是否拆成独立批量 Persister 留给阶段 7 的容量数据决定。

- `received_ack` 只代表进入实时层，Agent 不得删除本地事件。
- `durable_ack` 代表 PostgreSQL 已提交，Agent 才能删除对应 SQLite Outbox 行。
- PostgreSQL 通过 `UNIQUE(run_id, agent_sequence)` 去重。
- 高频文本 delta 允许有界合并；Item、Tool、最终结果和终态必须持久化。

### 4. 终态由 PostgreSQL 水位保护

Run 使用 `running -> finalizing -> completed`。只有 `persisted_through_sequence >= final_sequence` 时才能进入 `completed`。Redis 中观察到 terminal event 不能单独使 Run 完成。

### 5. Redis 故障通过协议重放恢复

Redis 丢失时，Run Server 从 PostgreSQL 获取持久基线；Agent 对未收到 `durable_ack` 的 SQLite Outbox 在同一连接内定时重放，并在重连时重放。显式按服务端水位发送 `run.replay` 留到阶段 7。

Agent 离线时只能展示 PostgreSQL 已持久部分，并明确标记 `waiting_agent`。本阶段不实现周期 Reconciler；Agent 重连握手必须主动上报未确认 Run 与序号并完成对账。

### 6. 多端读取采用两层 SSE

- Session SSE 只传递同一 Session 内的 `run.created`、状态、完成和 Agent 在线变化；Bootstrap 进度由 Sync Job HTTP 接口读取。
- Run SSE 传递单个 Run 的 item、delta 和 tool 增量。
- HTTP 快照和游标是首次加载及断线恢复基础。
- iPhone 与 Web 只调用 Run Server，不直接访问 Redis、PostgreSQL 或 Agent。

### 7. 客户端可触发异步 Bootstrap Sync

客户端通过 API 创建 `sync_job + PostgreSQL Outbox`。Dispatcher 将 `bootstrap.start` 发送给 Agent，Agent 分批读取 Codex Thread/Turn 并上传。Run Server 提交每批后再确认；只有仍活跃 Run 的事件才重建 Redis Stream，已完成历史只保存在 PostgreSQL。

当前单 Agent Runtime 全局同时只有一个活跃同步；客户端退出不取消；旧同步快照不得覆盖更新的实时数据。Bootstrap Sync 不等同于 Reconciler。

### 8. Redis 首期部署为一个隔离命名空间的高可用服务

首期只使用 `runtime:result:*`、固定 `runtime:presence:agent`、`runtime:notify:session:*`，通过 ACL、Key 前缀和独立连接池隔离职责。Redis DB 编号不作为隔离机制。缓存、限流或更多通知结构只有出现可测量需求并补充 ADR 后才引入。

## 数据权威矩阵

| 事实 | 权威来源 | 可恢复副本或加速层 |
| --- | --- | --- |
| Run 身份、命令接受、生命周期 | PostgreSQL | 无长期副本 |
| 待投递命令 | PostgreSQL Outbox | Redis 仅唤醒 |
| 未持久实时结果 | Mac Agent SQLite Outbox | Redis Result Stream |
| 已持久事件、终态、最终结果 | PostgreSQL | Redis 活跃期副本 |
| Agent presence | Redis TTL | PostgreSQL 可保存最后观测时间但不表示实时在线 |
| Bootstrap Sync 状态和批次检查点 | PostgreSQL | 无 Redis 副本 |

## 结果

收益：

- iPhone 与 Web 可以独立连接和恢复，同时查看同一 Session/Turn。
- Redis-first 保留流式低延迟，PostgreSQL 与 Agent SQLite 共同给出明确的持久边界。
- API、Dispatcher、WebSocket Relay、Result Persister 和 SSE 可以独立扩展。
- Redis 重启不会丢失已提交终态，也不会迫使系统盲目重复启动 Codex Turn。
- 初始化与实时链路使用相同的持久命令和批次确认原则。

代价：

- 系统采用至少一次投递，所有命令和事件消费者都必须幂等。
- 需要维护 PostgreSQL、Redis、Agent SQLite 三处水位及明确 ACK 语义。
- 需要限制 Redis Stream、Agent Outbox、SSE 连接和事件保留容量。
- 短暂存在“Redis 已收到但 PostgreSQL 尚未完成”的状态，界面必须表达 `finalizing`、`recovering` 和 `waiting_agent`。
- 跨五个独立仓库的协议、fixtures 和发布顺序必须受控。

## 被拒绝的方案

### 客户端直接通过 WebSocket 查看结果

拒绝作为目标方案。它会把客户端连接和 Agent 传输耦合，增加多端订阅、代理兼容、鉴权与恢复状态机复杂度。客户端的单向更新更适合 SSE，命令继续用 HTTPS。

### Redis 同时作为唯一命令队列和唯一结果来源

拒绝。Redis 的裁剪、TTL、故障转移和误操作不应决定 Run 是否存在或是否完成。命令接受与终态必须有 PostgreSQL 事务边界。

### 所有细粒度 delta 都先同步写 PostgreSQL

拒绝作为当前目标。它简化正确性但会把首 token 延迟和写放大置于同步路径。采用 Agent SQLite + Redis Result Stream + 异步持久化，在不牺牲可恢复性的前提下降低实时延迟。

### 通过完整查询接口轮询事件

拒绝作为新的事件传输方案。轮询 `/sessions/{id}/runs?after_session_sequence=` 只能返回重复的 Run 摘要，无法高效承载细粒度 delta。非 SSE 场景使用独立的增量 `events:poll` 契约；它与 Session/Run SSE 共享 PostgreSQL 权威事件和 sequence。完整查询接口只用于快照恢复。

### 首阶段引入周期 Reconciler

暂不采用。它会增加扫描、租约判断和错误重跑风险。第一阶段先用协议重连、持久水位和 SQLite 重放闭合已知故障；后续只有实际故障证据表明需要无人值守扫描时再写独立 ADR。

## 阶段 7 待固化参数

- [x] Runtime 使用独立 `runtime` Schema，当前开发角色和 migration 已落地。
- [ ] Prompt、Response、Diff 和工具输出的加密、保留与删除策略。
- [ ] Redis Stream 保留窗口和内存预算。
- [ ] Agent SQLite Outbox 上限、磁盘不足与损坏恢复策略。
- [ ] SSE 并发限制、心跳、游标过期和慢消费者策略。
- [x] 当前 Mac Agent 保持全局单活 Turn；更细串行键需独立需求再设计。
- [x] Runtime OpenAPI、协议类型和发布顺序已经落地；iPhone 兼容矩阵在阶段 8 补齐。

## 实现状态

- [x] 架构方向与 ADR 已接受。
- [x] 阶段 0 最小数据与接口契约已落地，生产容量参数留阶段 7。
- [x] PostgreSQL Runtime Core 已实现。
- [x] Dispatcher 与 Agent Durable Run 已实现。
- [x] Redis Result Stream、SQLite Outbox 和集成式 Runtime Event Persister 已实现。
- [x] Session SSE 与 Run SSE 已实现。
- [x] Bootstrap Sync 已实现。
- [x] mobile-web 真实通信和端到端主验收已通过。
- [ ] 故障注入、升级和回退验收已通过。
- [ ] iPhone 已在最后阶段迁移到 HTTPS + SSE。
