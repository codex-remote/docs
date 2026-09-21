---
title: Run Server JSON 轮询接口规范
date: 2026-08-25
updated: 2026-08-25
tags:
  - ai-coding-remote
  - runtime
  - api
  - polling
  - protocol
status: current
implementation_status: implemented-local
api_version: runtime-v1-poll
production_readiness: not-applicable
related:
  - "[[Runtime JSON 轮询传输架构]]"
  - "[[Run Server HTTPS 与 SSE 接口规范]]"
  - "[[ADR-009 Runtime JSON 轮询作为非 SSE 兼容传输]]"
---

# Run Server JSON 轮询接口规范

> [!note] 本地实现状态
> 两个 `events:poll` 路由已进入 `relay-server/apifox/openapi.json`、Gateway allowlist 和自动化契约测试。生产限流、Cursor 过期和公网边缘行为仍待验收。

## 1. 适用范围

本文只定义 Session/Run 增量事件的普通 JSON 长轮询。现有 SSE 接口继续保留，使用相同的事件类型、sequence 和终态语义。普通资源查询、创建、取消、Bootstrap 和源码读取仍使用原 Runtime v1 HTTP 接口。

浏览器请求路径仍然经过 Mobile Web Gateway；Run Server 负责认证、Scope 和资源授权。客户端不得直接连接 Redis/Valkey 或 PostgreSQL。

## 2. 接口总表

| 方法 | 路径 | 用途 | 状态 |
| --- | --- | --- | --- |
| `GET` | `/v1/runtime/sessions/{session_id}/events:poll` | 读取 Session sequence 之后的事件 | implemented-local |
| `GET` | `/v1/runtime/runs/{run_id}/events:poll` | 读取 Run agent sequence 之后的事件 | implemented-local |

两条路径都只读、可安全重试，不接受 Request Body。

## 3. 通用请求参数

| 参数 | 必填 | 默认 | 上限 | 说明 |
| --- | --- | ---: | ---: | --- |
| `after` | 否 | `0` | 由有符号 64 位 sequence 限制 | 只返回大于该游标的事件 |
| `wait_ms` | 否 | `15000` | `15000` | 没有事件时最多等待时长；`0` 表示只查一次 |
| `limit` | 否 | `100` | `100` | 单次最多返回事件数 |

服务端必须拒绝负数、无法解析或超过上限的参数，返回 `400 POLL_QUERY_INVALID`。资源 ID 继续使用 Runtime v1 的路径校验和授权规则。

请求 Header：

```text
Authorization: Bearer <short-lived-access-token>
Accept: application/json
X-Request-Id: <optional-client-id>
```

## 4. 成功响应

统一 Envelope：

```json
{
  "success": true,
  "data": {
    "events": [],
    "next_cursor": 0,
    "has_more": false,
    "timed_out": true,
    "terminal": false,
    "status": "running"
  },
  "meta": { "schema_version": 1 }
}
```

字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `events` | array | 按 sequence 升序排列的规范化 RuntimeEvent |
| `next_cursor` | integer | 本次已返回的最后 sequence；无事件时等于请求 `after` |
| `has_more` | boolean | 仍有事件未在本批返回，客户端应立即用 `next_cursor` 请求下一批 |
| `timed_out` | boolean | 等待窗口结束且没有新事件；不是业务失败 |
| `terminal` | boolean | 资源已经进入终态，客户端可以释放轮询 |
| `status` | string | 当前 Run 状态；Session 轮询可省略或返回当前关联状态 |

事件对象必须与 SSE 的事件数据保持语义等价，至少包含：

```json
{
  "id": "45",
  "type": "assistant.delta",
  "run_id": "run_01...",
  "delta": "hello"
}
```

`id` 是字符串形式的 sequence；`type` 和其余字段沿用 Runtime v1 事件规范。Run 事件必须包含 `run_id`。Session 事件至少包含 `session_id`、`run_id`（如果事件关联 Run）和状态变化字段。

## 5. 返回规则

1. 服务端先从 PostgreSQL 查询大于 `after` 的已持久化事件。
2. 查到事件后立即返回，最多返回 `limit` 条。
3. 没有事件且 `wait_ms > 0` 时，等待 durable notification 或等待窗口结束。
4. 被通知唤醒后必须重新从 PostgreSQL 查询，不能直接把 Redis/Valkey 尚未确认持久化的事件作为响应。
5. 等待窗口结束返回空数组、`next_cursor=after`、`timed_out=true`、HTTP `200`。
6. 如果请求期间资源进入终态，即使没有新增事件也返回当前 `status` 和 `terminal=true`。
7. 客户端不得把 `timed_out=true` 解释为 Run 失败或取消。

## 6. 错误响应

| HTTP | code | retryable | 场景 |
| ---: | --- | --- | --- |
| 400 | `POLL_QUERY_INVALID` | false | 参数无效或超限 |
| 401 | `AUTH_REQUIRED` / `AUTH_INVALID` | true | Access Token 缺失、过期或撤销 |
| 403 | `RUNTIME_SCOPE_REQUIRED` | false | 缺少 `runtime:read` |
| 404 | `SESSION_NOT_FOUND` / `RUN_NOT_FOUND` | false | 资源不存在 |
| 410 | `CURSOR_EXPIRED` | true | 未来启用事件保留窗口后游标过期 |
| 429 | `POLL_RATE_LIMITED` | true | 客户端或设备超过并发/频率限制 |
| 500 | `RUNTIME_INTERNAL` | true | 未分类服务故障 |
| 503 | `POSTGRES_UNAVAILABLE` | true | 权威存储不可用；通知层不可用时优先降级为有界 PostgreSQL 查询 |

错误 Envelope 遵循 Runtime v1，不能把超时空响应包装成错误。

Redis/Valkey 通知失败不应单独阻断轮询。服务端可以退化为在 `wait_ms` 窗口内做受限的 PostgreSQL 重查；只有 PostgreSQL 本身不可用时才返回 `503`。

## 7. 交付与兼容

- 新接口不改变已有 `GET /.../events` SSE 路径。
- 新增字段只能向后兼容地添加；修改 sequence、事件类型或终态含义必须升级协议版本。
- 实现前需要同步更新 `relay-server/apifox/openapi.json`、`mobile-web/gateway/contract_v1.go` 和兼容记录；在此之前这些文件不得声称已公开新路由。
- Fixtures 至少覆盖空超时、单事件、多事件、跨批、重复 cursor、终态、参数错误和权限错误。
- 事件响应的日志只记录资源哈希、cursor、批大小、等待耗时和结果码，不记录 Prompt 或事件正文。
