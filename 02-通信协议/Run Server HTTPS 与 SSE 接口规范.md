---
title: Run Server HTTPS 与 SSE 接口规范
date: 2026-08-14
updated: 2026-08-22
tags:
  - ai-coding-remote
  - runtime
  - api
  - sse
  - apifox
  - implemented
aliases:
  - Runtime 前后端接口规范
  - Run Server API 规范
status: current
implementation_status: implemented
api_version: runtime-v1
apifox_status: local-contract-updated-sync-pending
completed: false
production_readiness: blocked
related:
  - "[[可靠多端会话与运行时控制面升级方案]]"
  - "[[Runtime PostgreSQL 数据表规范]]"
  - "[[Runtime SQLite 与 Valkey 数据结构规范]]"
  - "[[WebSocket 长连接模型]]"
  - "[[可靠多端运行时分阶段开发计划]]"
  - "[[Mobile Web Gateway 与 Runtime 鉴权架构]]"
  - "[[Run Server JSON 轮询接口规范]]"
---

# Run Server HTTPS 与 SSE 接口规范

> [!success] Runtime v1 Current
> 本文 13 个 Runtime operation 和 4 个公开 Auth operation 已在 `relay-server` 实现；本地 OpenAPI 已更新，Apifox 远端同步待单独执行。mobile-web 已接入 Gateway、配对和自动 Refresh。

> [!warning] 公网生产边界仍未完成
> Gateway 与 Runtime Auth 已实现；限流、Cursor 过期 `410`、多实例、稳定域名、TLS 和公网恢复演练仍未完成，因此 `completed` 保持 `false`。

> [!note] HTTP 与 HTTPS
> 局域网入口当前是 Gateway 上的 HTTP，Gateway 到 Run Server 也是 Loopback HTTP。未来公网由边缘终止 HTTPS，浏览器路径和内部协议不变；公网必须启用 `Secure` Refresh Cookie。

> [!important] 契约来源顺序
> 人工设计意图记录在本文；机器可执行事实以 `relay-server` 中版本化 OpenAPI、SSE JSON Schema 和 Fixtures 为准；Apifox 是同步后的协作与调试视图，不是唯一源。三者不一致时阶段验收失败。

> [!note] SSE 保留与非 SSE 传输
> SSE 是当前已实现的实时传输，不能因增加兼容通道而删除或改义。非 SSE 的 JSON 增量轮询另见 [[Run Server JSON 轮询接口规范]]；两个 `events:poll` 路径已进入本地 OpenAPI、Gateway allowlist 和自动化测试，生产限流与边缘验收仍待完成。

## 1. 客户端边界

- `mobile-web` 是阶段 1-7 的主要前后端联调和人工验证客户端。
- Mobile Web Gateway 是当前浏览器入口；它透明代理本文契约，不创建第二套 Runtime API，也不改变响应 Envelope 或 SSE 事件。
- iPhone 在阶段 8 最后迁移，复用已经由 mobile-web 验证的同一 HTTPS/SSE 语义。
- 客户端只能访问 Run Server HTTPS/SSE，不直接访问 Redis、PostgreSQL、Mac Agent WSS 或 Admin API。
- UI 组件不拼接 URL；`mobile-web` 通过 `RuntimeClient` Adapter 消费契约。
- 关闭 SSE、切换页面或刷新浏览器不等于取消 Run。

### 1.1 Gateway 接入合同

- 局域网目标 Base URL 为 `http://<mac-lan-ip>:18774`；未来公网使用稳定 HTTPS Origin。
- Mobile Web 使用 `/v1/runtime/*` 相对路径。浏览器不得推导或直接连接 `:18775`。
- Gateway 只代理本文登记的 Runtime Path；未知路径、Agent WSS、旧 App WSS、`/status` 和 Auth Control 均在本地拒绝。
- `Authorization`、`Last-Event-ID`、`Idempotency-Key`、`X-Request-Id`、状态码、Content Type 和流式 Body 必须保持语义等价。
- Gateway 错误使用 Gateway 自身稳定错误码，不伪装成 Runtime 业务错误；Apifox 仍以 Run Server 机器契约为准，不为透明代理复制接口。
- Gateway 当前暴露契约标识为 `run-server-v1`，由 `/gateway/healthz` 返回；新增上游接口必须显式通过 Gateway allowlist 审计。

## 2. 统一 HTTP 约定

| 项目 | 约定 |
| --- | --- |
| Base Path | `/v1/runtime`；阶段 0 冻结后不可在 v1 内改义 |
| 认证 | `Authorization: Bearer <short-lived-token>`；示例和日志不得包含真实 Token |
| Content Type | JSON 请求/响应使用 `application/json`；SSE 使用 `text/event-stream` |
| 时间 | RFC 3339 UTC |
| ID | 服务端资源 ID 使用带前缀 ULID；客户端不得自行构造服务端 ID |
| 写入幂等 | 创建 Run、创建 Session、Bootstrap 等写入使用 `Idempotency-Key` |
| Trace | 客户端可传 `X-Request-Id`；服务端在响应 Header 返回相同值或生成新值 |
| 分页 | Cursor 模式，响应包含 `next_cursor`；不提供无限 Offset |
| 兼容 | 新增可选字段可在同一版本演进；删除、改义或必填变化提升主版本 |

成功响应 Envelope：

```json
{
  "success": true,
  "data": {},
  "meta": { "schema_version": 1 }
}
```

错误响应 Envelope：

```json
{
  "success": false,
  "error": {
    "code": "RUN_NOT_FOUND",
    "message": "未找到指定的运行记录。",
    "retryable": false,
    "detail": {}
  },
  "meta": { "schema_version": 1 }
}
```

`detail` 只能包含 allowlist 字段，不返回 SQL、堆栈、Token、路径或内部 Redis Key。

## 3. 接口总表

| 方法 | 路径 | 用途 | 阶段 | 状态 |
| --- | --- | --- | --- | --- |
| `GET` | `/v1/runtime/healthz` | Runtime 进程健康，不泄露依赖秘密 | 1 | 已实现 |
| `GET` | `/v1/runtime/projects` | 当前 Mac Agent 的项目列表 | 1/6 | 已实现 |
| `GET` | `/v1/runtime/sessions` | Session 列表；mobile-web 用于发现其他客户端新建的 Session | 1 | 已实现 |
| `POST` | `/v1/runtime/sessions` | 建立尚无 Codex Thread 的 Session | 1 | 已实现 |
| `GET` | `/v1/runtime/sessions/{session_id}` | Session 快照与 Run 摘要 | 1/4 | 已实现 |
| `GET` | `/v1/runtime/sessions/{session_id}/runs` | Run 列表 | 1/4 | 已实现 |
| `POST` | `/v1/runtime/sessions/{session_id}/runs` | 幂等创建 Run | 1 | 已实现 |
| `GET` | `/v1/runtime/runs/{run_id}` | Run 状态与持久事件快照 | 1/3 | 已实现 |
| `POST` | `/v1/runtime/runs/{run_id}/cancel` | 幂等记录取消意图 | 2 | 已实现 |
| `GET` | `/v1/runtime/sessions/{session_id}/events` | Session SSE | 4 | 已实现 |
| `GET` | `/v1/runtime/runs/{run_id}/events` | Run SSE | 4 | 已实现 |
| `POST` | `/v1/runtime/bootstrap-syncs` | 幂等触发当前 Mac Agent 初始化 | 5 | 已实现 |
| `GET` | `/v1/runtime/bootstrap-syncs/{sync_id}` | 初始化状态与检查点 | 5 | 已实现 |

### 3.1 Auth 接口

| 方法 | 路径 | 凭证 | 状态 |
| --- | --- | --- | --- |
| `POST` | `/v1/auth/pairing-grants:exchange` | 一次性 Pairing Code | 已实现 |
| `POST` | `/v1/auth/tokens:refresh` | HttpOnly Refresh Cookie | 已实现 |
| `POST` | `/v1/auth/sessions/current:revoke` | HttpOnly Refresh Cookie | 已实现 |
| `GET` | `/v1/auth/me` | Bearer Access Token | 已实现 |

三个 Auth `POST` 都要求 `X-CodexRemote-Request: 1`。Access Token 默认 15 分钟，Refresh Token 默认 30 天并每次使用后轮换；重放已使用 Refresh Token 会撤销整个会话。配对授权只能由 Loopback Auth Control 创建，Control 路由不属于公开 OpenAPI，也不经过 Gateway。

## 4. `GET /projects`

用途：读取当前唯一已配置 Mac Agent 上报或 Bootstrap 导入的项目元数据。响应 `data` 除 `items/next_cursor` 外可包含 `agent_presence` 和 `agent_last_seen_at`；不建立 Agent 业务资源。

响应字段：`project_id`、`display_name`、`next_cursor`。不得返回 Mac 本地绝对路径。

## 5. `GET /sessions`

查询参数：

| 参数 | 必填 | 说明 |
| --- | --- | --- |
| `project_id` | 否 | 按 Project 过滤 |
| `cursor` | 否 | 上一页返回值 |
| `limit` | 否 | 默认值和上限在阶段 0 冻结 |

每个条目至少返回：`session_id`、`project_id`、`codex_thread_id`、`title`、`last_session_sequence`、`latest_run` 摘要和 `updated_at`。

## 6. `POST /sessions`

Header：`Idempotency-Key` 必填。

请求：

```json
{
  "project_id": "project_01...",
  "title": "订单接口优化"
}
```

成功：`201 Created`，返回 `session_id` 和 `last_session_sequence=0`。相同幂等键、相同请求返回原 Session；相同键不同请求返回 `409 IDEMPOTENCY_CONFLICT`。

## 7. `GET /sessions/{session_id}`

用途：首次进入、刷新、SSE Cursor 过期时取得权威快照。

响应至少包含：

- Session 元数据和 `snapshot_session_sequence`。
- 当前 Agent presence。
- 按 Cursor 分页的已持久 Turn 摘要。
- 活跃 Run 摘要及每个 Run 的 `persisted_through_sequence`。
- `sync_job` 当前摘要。
- `truncated` 和后续分页 Cursor。

快照不能假装包含 Redis 尚未持久化的完整增量；客户端连接 Run SSE 后补齐活跃 overlay。

## 8. `GET /sessions/{session_id}/runs`

用途：读取 Run 摘要和快照恢复，不承担事件增量传输。

查询参数：`after_session_sequence`、`cursor`、`limit`、`status`。响应返回 Run 摘要和当前 `session_sequence`。

该接口不会替代 Session SSE，也不作为新的事件轮询接口。Session/Run 事件的非 SSE 降级统一使用 [[Run Server JSON 轮询接口规范]] 中的两个 `events:poll` 路径；本文接口只用于首次加载、快照恢复和游标重建。

## 9. `POST /sessions/{session_id}/runs`

Header：`Idempotency-Key` 必填。

请求：

```json
{
  "prompt": "检查订单接口的并发问题，并运行相关测试。",
  "client_context": {
    "locale": "zh-CN"
  }
}
```

成功：仅在 PostgreSQL 的 Run（含幂等键）、Session Event 和 Command Outbox 同一事务提交后返回：

```json
{
  "success": true,
  "data": {
    "run_id": "run_01...",
    "session_id": "session_01...",
    "status": "queued",
    "session_sequence": 41,
    "codex_thread_id": null,
    "codex_turn_id": null,
    "created_at": "2026-08-14T08:01:00Z"
  },
  "meta": {
    "schema_version": 1
  }
}
```

HTTP 状态使用 `202 Accepted`。`run_id` 是客户端凭证，不等于 Codex ID。

## 10. `GET /runs/{run_id}`

用途：读取 Run 快照和恢复水位。

至少返回：`status`、`state_version`、Codex ID 映射、`persisted_through_sequence`、`final_sequence`、最终结果、错误和时间字段。`recovering`、`waiting_agent` 直接使用 Run `status` 表达，不再返回重复的 `recovery_state`。

读取规则：

- 运行中：PostgreSQL 生命周期和持久基线 + Redis 活跃摘要。
- `completed`：完整结果只读 PostgreSQL。
- Redis 缺失且 Agent 在线：Run 状态进入 `recovering`。
- Redis 缺失且 Agent 离线：Run 状态进入 `waiting_agent`，只返回持久部分。

## 11. `POST /runs/{run_id}/cancel`

请求 Body 固定为空 JSON `{}`，不需要 `Idempotency-Key`。Run 的 `cancel_requested_at` 与唯一 `run.cancel` Command Outbox 已保证重复请求返回同一状态。

成功只表示取消意图已写 PostgreSQL并进入 Command Outbox，不表示 Codex 已停止。响应包含 `cancel_requested=true` 和当前 Run 状态。

## 12. Session SSE

路径：`GET /v1/runtime/sessions/{session_id}/events`。

请求 Header：

- `Accept: text/event-stream`
- `Last-Event-ID: <session_sequence>`，首次连接可省略。
- `Authorization: Bearer ...`

事件示例：

```text
id: 41
event: run.created
data: {"schema_version":1,"session_id":"session_01...","run_id":"run_01...","status":"queued","occurred_at":"2026-08-14T08:01:00Z"}

```

允许事件：

| `event` | 用途 |
| --- | --- |
| `run.created` | 另一客户端创建新 Run |
| `run.status.changed` | Run 低频状态变化 |
| `run.completed` | 最终结果已满足 PostgreSQL 完成门禁 |

Session SSE 不发送 assistant delta、命令输出或工具明细。

## 13. Run SSE

路径：`GET /v1/runtime/runs/{run_id}/events`。

`Last-Event-ID` 使用 `agent_sequence`。Run Server 先读取 PostgreSQL 缺口，再接入 Redis 活跃增量，并在切换点去重。

事件示例：

```text
id: 128
event: item.delta
data: {"schema_version":1,"run_id":"run_01...","agent_sequence":128,"item_id":"item_01...","delta":"正在检查并发访问...","occurred_at":"2026-08-14T08:01:02Z"}

```

当前允许事件：`run.accepted`、`turn.started`、`assistant.delta`、`item.started`、`item.delta`、`item.completed`、`turn.completed`、`turn.failed`、`turn.interrupted`。

规则：

- SSE `id` 必须等于业务 `agent_sequence`，不能使用 Redis Stream ID。
- 重复 ID 的相同事件由客户端幂等忽略；相同 ID 内容冲突必须中止并上报。
- 15-30 秒无业务事件时发送 SSE 注释心跳，具体值阶段 0 冻结。
- 慢客户端可被断开，但不能拖慢持久化；mobile-web 使用最后已应用 Cursor 自动重连。
- `turn.completed` 只有 PostgreSQL 满足 `persisted_through_sequence >= final_sequence` 后发送。

## 14. Bootstrap Sync API

### `POST /bootstrap-syncs`

Header：`Idempotency-Key` 必填。若已有 `queued/running` Job，返回已有 `sync_id`，不创建第二个。

成功响应 `202 Accepted`：`sync_id`、`status`、`created_at`。

### `GET /bootstrap-syncs/{sync_id}`

返回：`status`、`snapshot_id`、`last_committed_batch_no`、`item_count`、错误、`created_at` 和 `updated_at`。

客户端关闭页面不取消 Sync。首期客户端通过本接口有界轮询进度；Bootstrap 不属于单个 Session，因此不把进度复制到各 Session SSE。

## 15. 错误码基线

| HTTP | 错误码 | 可重试 | 说明 |
| --- | --- | --- | --- |
| 400 | `REQUEST_INVALID` | 否 | Schema 或字段校验失败 |
| 401 | `AUTH_INVALID_CREDENTIAL` / `AUTH_ACCESS_EXPIRED` / `AUTH_SESSION_REVOKED` | 条件 | Access Token 缺失、无效、过期或会话撤销 |
| 401 | `AUTH_CREDENTIAL_EXPIRED` / `AUTH_REFRESH_REPLAY` | 否 | 配对/Refresh 过期，或检测到 Refresh 重放 |
| 403 | `AUTH_SCOPE_REQUIRED` / `RESOURCE_FORBIDDEN` | 否 | Scope 不足或无权访问资源 |
| 404 | `AGENT_NOT_FOUND` / `SESSION_NOT_FOUND` / `RUN_NOT_FOUND` | 否 | 资源不存在或按安全策略隐藏 |
| 409 | `IDEMPOTENCY_CONFLICT` | 否 | 相同幂等键对应不同请求 |
| 409 | `RUN_CONCURRENCY_CONFLICT` | 条件 | 同一串行范围已有活跃 Run |
| 410 | `CURSOR_EXPIRED`（规划） | 是 | 重新取快照后恢复；当前尚未实现 |
| 422 | `RUN_STATE_INVALID` | 否 | 当前状态不允许操作 |
| 429 | `RATE_LIMITED`（规划） | 是 | 尊重 `Retry-After`；当前尚未实现 |
| 503 | `AGENT_UNAVAILABLE` | 是 | Agent 离线或不可接收新命令 |
| 503 | `RUNTIME_DEPENDENCY_UNAVAILABLE` | 是 | PostgreSQL/Valkey 等依赖降级 |

## 16. mobile-web 主要验收矩阵

| 场景 | 前端动作 | 服务端事实 | 必须观察到 |
| --- | --- | --- | --- |
| 创建 Run | POST + Idempotency-Key | PostgreSQL 已提交 | 重试返回同一 `run_id` |
| 双窗口发现 | A 创建，B 订阅 Session SSE | `session_events` 有新序号 | B 无刷新出现 Run |
| 流式结果 | 两端订阅 Run SSE | Redis 活跃流 + PG 水位 | 顺序一致、无重复 |
| 刷新页面 | 重新取 Session/Run 快照 | PG 基线存在 | 从 Cursor 补齐而非清空 |
| Redis 清空 | 保持页面观察 | PG + Agent SQLite 恢复 | 显示 recovering，随后补齐 |
| Agent 离线 | 查询运行中 Run | 只有 PG 持久部分 | 显示 waiting_agent，不假完成 |
| 取消 | POST cancel | 取消意图已持久化 | UI 区分 requested 与 canceled |
| Cursor 过期 | 旧 Last-Event-ID 重连 | 服务端返回恢复信号 | 客户端重新快照后继续 |

## 17. Apifox 同步规范

现有 Apifox 项目：`CodexRemote`，项目 ID `8693796`。Runtime v1 与原 Relay 状态/WSS 接口都从 `relay-server/apifox/openapi.json` 和 WebSocket 定义同步。

> [!success] Apifox Current 同步状态
> 2026-08-14 已同步 13 个 Runtime operation，创建 13 个 endpoint case、20 个 Schema 和 3 个复用响应；远端回读确认无导入错误。`/ws/agent` 资源 ID `3884669`、`/ws/app` 资源 ID `3884670` 也已回读一致。

### Runtime Current 资源

- [x] Runtime HTTP/SSE OpenAPI 已导入并回读。
- [x] Health `502032734`、Projects `502032735`、Sessions list/create `502032736`/`502032737`。
- [x] Session snapshot/runs/create-run `502032738`/`502032739`/`502032740`。
- [x] Run snapshot/cancel `502032741`/`502032742`。
- [x] Session SSE/Run SSE `502032743`/`502032744`。
- [x] Bootstrap create/status `502032745`/`502032746`。

### 每次接口变更门禁

1. 先更新本文中的业务语义和字段说明。
2. 更新 `relay-server` OpenAPI、SSE JSON Schema 和 Fixtures。
3. 运行本地 OpenAPI/Schema 校验和服务端契约测试。
4. `mobile-web` 对同一 Fixtures 做解码与交互测试。
5. 使用仓库 Apifox 同步工具执行远端只读检查。
6. 只有确认目标模块和差异后才同步，随后再次读取验证。
7. 在阶段验收记录中填写 Apifox 项目、模块、同步时间、资源 ID、源文件 commit 和验证结果。

### Current 变更门禁

以下条件全部满足后才能继续同步 Current：

- Run Server 路由和 Store 已实现并通过集成测试。
- mobile-web 已用真实接口完成对应交互测试。
- OpenAPI、SSE Schema、Fixtures、本文和实现无差异。
- 人工在 Apifox 选定环境完成请求/SSE 验证。
- 兼容矩阵和回退策略已经记录。

> [!warning] 未实现能力边界
> Runtime/Auth 本地 OpenAPI 已更新。限流、Cursor 过期、多实例和公网能力不得预先加入成功契约。Gateway 是部署入口，不改变 13 个 Runtime operation 的业务 Schema；4 个 Auth operation 是独立路由组。

## 18. 接口变更记录模板

```markdown
### Runtime API 契约变更

- 接口与版本：
- 业务原因：
- Obsidian 章节：
- OpenAPI / SSE Schema 文件：
- Fixtures：
- relay-server commit：
- mobile-web commit：
- 兼容性：兼容 / 破坏性
- Apifox 项目与模块：
- Apifox 资源 ID：
- 同步时间：
- 同步前后校验：
- 人工验证结果：
```

缺少任一项时，接口视为未同步、未完成。
