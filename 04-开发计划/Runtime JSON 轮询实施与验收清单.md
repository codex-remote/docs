---
title: Runtime JSON 轮询实施与验收清单
date: 2026-08-25
updated: 2026-08-26
tags:
  - ai-coding-remote
  - runtime
  - polling
  - implementation-plan
status: active
implementation_status: local-implementation-complete
related:
  - "[[ADR-009 Runtime JSON 轮询作为非 SSE 兼容传输]]"
  - "[[Runtime JSON 轮询传输架构]]"
  - "[[Run Server JSON 轮询接口规范]]"
  - "[[可靠多端运行时分阶段开发计划]]"
---

# Runtime JSON 轮询实施与验收清单

> [!note] 边界说明
> 本清单只跟踪边缘无关的 JSON Poll 传输。接口、Gateway allowlist 和前端 Adapter 已完成本地实现；局域网压力与恢复项仍待验收。公网入口不属于本清单。

## 阶段 0：契约冻结

- [x] 确认 `events:poll` 两个路径、`after`、`wait_ms`、`limit` 和响应 Envelope。
- [x] 确认 Session/Run 事件对象与 SSE 事件语义完全等价。
- [x] 确认 `terminal`、`timed_out`、`has_more` 和 `next_cursor` 的边界。
- [x] 为空超时、重复 cursor、跨批、终态和错误建立 JSON Fixtures。
- [x] 明确事件序号保留窗口和未来 `410 CURSOR_EXPIRED` 的兼容策略。

## 阶段 1：`relay-server` 后端

- [x] 增加 Session/Run 持久事件的有序增量读取。
- [x] 增加 `wait_ms`、`limit`、资源 ID 和 cursor 参数校验。
- [x] 实现 request context 取消和有界等待。
- [x] 在 PostgreSQL 持久化成功后发出 Session/Run 通知。
- [x] 通知唤醒后重新查询 PostgreSQL，禁止直接返回未 durable 的 Redis 事件。
- [x] Redis/Valkey 通知不可用时退化为受限 PostgreSQL 重查。
- [x] 增加基础认证、Scope、资源隔离和参数错误测试；429/压力测试仍待补充。
- [x] 更新 `apifox/openapi.json`、Fixtures、`CHANGELOG.md` 和协议说明。

## 阶段 2：`mobile-web` Gateway

- [x] 在 `gateway/contract_v1.go` 中显式加入两个新 GET 路径。
- [x] 保持同源、Bearer、Cookie、`X-Request-Id` 和 JSON Envelope 语义。
- [x] 验证未知 Runtime 路径仍然被拒绝。
- [x] 增加代理成功、上游错误、取消传播和响应头测试。
- [x] 更新 `docs/run-server-v1-compatibility.md`，继续沿用 `run-server-v1` 的增量兼容边界。

## 阶段 3：`mobile-web` 前端

- [x] 在 `RuntimeClient` 后实现 `SseTransport` 与 `JsonPollTransport`。
- [x] UI 仅消费统一的 `watchSession`/`watchRun` 事件流。
- [x] 为轮询请求设置独立于普通请求的 20-25 秒 timeout。
- [x] 实现 abort、401 Refresh、唯一资源控制器和现有重连循环。
- [x] 按 `(resource_id, sequence)` 幂等处理重复事件。
- [x] 更新连接检查器的 transport 状态。
- [x] 保持 SSE 解析器和现有 SSE 测试不变并继续通过。

## 阶段 4：局域网验收

- [ ] 新建 Session、创建 Run、连续 delta、工具事件和终态在轮询模式下与 SSE 一致。
- [ ] 强制丢弃响应、重复请求、页面刷新和 Session 切换，确认无事件丢失。
- [ ] 停止/恢复 Redis，确认实时性下降但 PostgreSQL durable 事件仍可恢复。
- [ ] 重启 Gateway 或 Run Server，确认 cursor 快照恢复和 Token Refresh。
- [ ] 长回答、多个活动 Run、长时间空闲、移动 Safari 后台恢复。
- [ ] 记录实际请求数、响应大小、数据库查询耗时、并发等待数和重连次数。

## 阶段 5：边缘入口边界

- [x] 保持边缘行为不进入 Runtime 业务层。
- [x] SSE 继续作为独立传输保留。
- [x] 否决不满足可靠性要求的临时随机域名入口，见 [[ADR-010 TryCloudflare Quick Tunnel 作为临时公网轮询入口]]。
- [ ] 未来稳定公网入口另行冻结架构、安全和验收范围。

## 发布门槛

只有阶段 0-4 全部完成，且 `relay-server` 与 `mobile-web` 的独立测试、契约记录和版本说明同步后，才能把 Poll 传输标记为完整局域网验收。公网放行需要独立验收，且不回写为 Runtime 事件契约的隐含条件。
