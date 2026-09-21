---
title: ADR-009 Runtime JSON 轮询作为非 SSE 兼容传输
date: 2026-08-25
tags:
  - ai-coding-remote
  - adr
  - runtime
  - polling
  - compatibility
aliases:
  - ADR-009
status: accepted
implementation_status: implemented-local
decision_date: 2026-08-25
related:
  - "[[Runtime JSON 轮询传输架构]]"
  - "[[Run Server JSON 轮询接口规范]]"
  - "[[Run Server HTTPS 与 SSE 接口规范]]"
  - "[[Mobile Web Gateway 与 Runtime 鉴权架构]]"
  - "[[ADR-007 PostgreSQL 权威运行时与 Redis 实时加速]]"
---

# ADR-009 Runtime JSON 轮询作为非 SSE 兼容传输

## 状态

已接受并完成本地代码实现。`relay-server`、Mobile Web Gateway 和 `mobile-web` 已提供轮询入口及自动化测试；真实局域网真机、长时间断线和任何公网边缘入口仍待单独验收。本文不决定任何 Cloudflare、域名或公网连接器方案。

## 背景

Runtime v1 当前通过同源 HTTP 和 Session/Run SSE 向 Mobile Web 提供事件。SSE 是已实现的实时传输，必须继续保留。部分只支持普通 HTTP 响应的边缘入口不能承载 SSE，但仍可转发有界的 JSON 请求/响应。

如果直接用固定间隔请求完整 Run 快照，会重复传输已经消费的事件，随着回答变长产生不必要的 PostgreSQL、Gateway 和网络开销。因此需要增加一个使用持久事件游标的非 SSE 传输，而不是复制一套业务状态模型。

## 决策

1. 在现有 Runtime 事件模型之上增加 JSON 长轮询传输；不删除、不改义现有 SSE 路径。
2. 新增独立的 `events:poll` HTTP GET 接口，返回有限批次的已持久化增量事件。
3. Session sequence 和 Run agent sequence 继续作为唯一游标；轮询与 SSE 使用相同事件类型、终态语义和恢复边界。
4. PostgreSQL 是轮询响应的权威事件来源。Redis/Valkey 只负责唤醒等待请求或提供实时加速，不得成为轮询客户端推进游标的唯一事实来源。
5. 事件投递采用至少一次语义。网络重试允许重复返回同一 sequence，客户端必须幂等应用事件。
6. 前端通过 `RuntimeClient` 后面的传输适配选择 `poll` 或 `sse`；UI 和业务事件处理不感知具体传输。
7. `wait_ms`、`limit`、客户端超时、并发和重试都有硬上限；任何请求不得无限挂起。
8. 新接口先更新语言无关设计文档和仓库内契约记录；实现完成前不得把它写入“已实现”的 OpenAPI 或 Gateway allowlist。

## 轮询请求边界

```text
Browser
  -> Mobile Web Gateway
  -> Run Server Auth + Runtime API
  -> PostgreSQL event tables

Redis/Valkey notification
  - only wakes a bounded request
  - never replaces PostgreSQL durability
```

单个活动 Session 通常占用一个 Session 轮询请求，单个活动 Run 占用一个 Run 轮询请求。响应返回或超时后，客户端才创建下一次请求；客户端不得为同一资源并发创建多个轮询循环。

## 被拒绝的方案

### 固定间隔的完整快照轮询

拒绝作为最终方案。它会重复返回历史事件，产生 O(事件总量 × 轮询次数) 的响应体，并把无事件时的等待转化为持续数据库查询。现有完整快照接口仍用于首次加载、恢复和游标不可用时的重建。

### 通过 JSON Chunked 或自定义流模拟 SSE

拒绝。分块响应仍然依赖边缘对长连接和响应 flush 的支持，无法解决“不支持 SSE”的入口限制，也会重新引入缓冲和超时差异。

### 把 Redis Stream 直接作为客户端事实源

拒绝。当前事件先进入实时层，再完成 PostgreSQL 持久化；进程故障可能使客户端看到尚未 durable 的事件。Redis 只能作为唤醒和加速层，返回前必须以 PostgreSQL 事件水位确认。

### 在 UI 中写两套事件处理逻辑

拒绝。SSE 与 JSON 轮询必须在 `RuntimeClient`/Transport Adapter 层收敛为同一 `RuntimeEvent` 输出，避免终态、工具状态和恢复逻辑漂移。

## 代价与风险

- 长轮询占用有限的 HTTP 请求和 Go goroutine；必须限制等待时长、并发和客户端重试。
- Redis 通知丢失时，下一次超时重连仍必须通过 PostgreSQL 游标恢复，不能丢数据。
- 网络响应丢失会造成重复批次；客户端幂等处理是必需条件。
- 非 SSE 传输的实时性由等待窗口和边缘网络决定，不能承诺 SSE 的低延迟表现。
- 新增接口需要同时审查 Run Server、Gateway allowlist、前端 Adapter、OpenAPI、Fixtures 和验收脚本。

## 验收门槛

本地接口、契约测试和两个传输适配已完成；断线长时间恢复、限流、Cursor 过期和长回答压力仍是后续验收门槛。任何公网边缘入口都需要独立决策，不作为本 ADR 的实现前提，也不回写 Poll 业务契约。
