---
title: Runtime PostgreSQL 数据表规范
date: 2026-08-14
tags:
  - ai-coding-remote
  - runtime
  - postgresql
  - data-model
  - implemented
aliases:
  - Run Server PostgreSQL 数据字典
status: current
implementation_status: implemented
schema_version: runtime-postgresql-v1
completed: false
production_readiness: blocked
related:
  - "[[可靠多端会话与运行时控制面升级方案]]"
  - "[[Runtime SQLite 与 Valkey 数据结构规范]]"
  - "[[Run Server HTTPS 与 SSE 接口规范]]"
  - "[[ADR-007 PostgreSQL 权威运行时与 Redis 实时加速]]"
  - "[[可靠多端运行时分阶段开发计划]]"
---

# Runtime PostgreSQL 数据表规范

> [!success] Runtime PostgreSQL v1 已实现
> `relay-server/internal/runtime/migrations/001_runtime.sql` 已落地本文 7 张业务表。2026-08-14 已通过真实 PostgreSQL 17 migration、20 路同幂等键并发和 Agent 端到端写入验证。

> [!warning] 生产治理未完成
> 表结构与事务主链已经实现，但 Prompt/Result 加密、数据库最小权限角色、明确保留期限和自动清理仍未冻结。因此本文不能作为“可直接公网生产”的完成证明；相关检查项保持未勾选。

## 1. 收敛原则

1. 只保存当前已确认链路所需的数据：Project、Session、Run、持久事件、命令投递和初始化检查点。
2. 不为多租户、通用任务系统、设备配对、密钥轮换、审计、标签、搜索或未知客户端预建表和字段。
3. Codex 的历史 Turn 统一映射为 `runs`，历史内容统一映射为 `run_events`；不再建立语义重复的 Turn、Item 表。
4. 幂等信息保存在实际创建的聚合记录上；不建立通用 `idempotency_records` 表。
5. Bootstrap 首期严格串行、逐批 durable ACK；只保存最新提交水位，不建立批次历史表。
6. PostgreSQL 是已接受命令、Run 状态、持久结果和初始化进度的权威来源；Redis 可全部丢失并重建。

## 2. 表清单

Runtime v1 只有以下 7 张业务表：

| 表 | 唯一职责 |
| --- | --- |
| `runtime.projects` | Agent 上可展示和可选择的项目 |
| `runtime.sessions` | 多端共同查看的会话聚合与 Session SSE 水位 |
| `runtime.runs` | 一次用户提交或一条导入历史 Turn 的状态与持久水位 |
| `runtime.session_events` | 跨客户端发现新 Run 和状态变化的持久事件 |
| `runtime.run_events` | Run 内按序持久化的输出事件 |
| `runtime.command_outbox` | PostgreSQL 提交后可靠投递给 Agent 的命令 |
| `runtime.sync_jobs` | 一次 Agent 历史初始化的当前状态与续传水位 |

`schema_migrations` 由迁移工具管理，不算业务表。

## 3. 通用约定

| 项目 | 约定 |
| --- | --- |
| Schema | 业务表统一放在 `runtime` schema |
| ID | 应用生成带类型前缀的 ULID，PostgreSQL 使用 `text` |
| 时间 | `timestamptz`，应用和数据库统一使用 UTC |
| JSON | 仅用于版本化事件或命令 Payload，不用 JSON 代替明确关系字段 |
| 序号 | Session 和 Run 内使用 `bigint` 单调递增，从 1 开始 |
| 删除 | v1 不提供业务级级联删除接口；保留策略在阶段 0 冻结后补充 migration |
| 更新 | 状态流转通过带旧状态条件的 `UPDATE`，禁止无条件覆盖 |

## 4. `runtime.projects`

| 字段 | 类型 | 空值/默认 | 约束与说明 |
| --- | --- | --- | --- |
| `project_id` | `text` | 非空 | 主键 |
| `display_name` | `text` | 非空 | 客户端可展示名称；不保存 Mac 绝对路径 |
| `last_seen_snapshot_id` | `text` | 可空 | 最近一个已 durable 提交批次看到该项目的快照 ID |
| `archived_at` | `timestamptz` | 可空 | 非空表示已从当前 Agent 项目目录消失；数据仍保留 |
| `created_at` | `timestamptz` | `now()` | 首次进入 Runtime 的时间 |
| `updated_at` | `timestamptz` | `now()` | 名称、可见性或实时目录刷新时间 |

Run Server 持久化 Agent `project.snapshot` 并恢复重新出现的项目。只有未续传的完整 Bootstrap 最终批次可以软归档本次快照缺失的 Project；中途失败、Agent 断连、持久水位续传和仍有活跃 Run 的项目均不得归档。Runtime API 默认只返回 `archived_at IS NULL` 的项目及其会话。当前 Relay 同时只允许一个已配置 Mac Agent，`project_id` 已足以确定归属，不在 PostgreSQL 重复保存 `agent_id`。

写入者：Run Server、Bootstrap Persister。读取者：Run Server。

## 5. `runtime.sessions`

Session 是 iPhone、mobile-web 以及未来客户端共同观察的会话身份。

| 字段 | 类型 | 空值/默认 | 约束与说明 |
| --- | --- | --- | --- |
| `session_id` | `text` | 非空 | 主键 |
| `project_id` | `text` | 非空 | 外键到 `projects.project_id` |
| `codex_thread_id` | `text` | 可空 | Agent 创建或 Bootstrap 识别后回填 |
| `title` | `text` | 可空 | 会话展示标题 |
| `last_session_sequence` | `bigint` | `0` | 已提交的最新 Session Event 序号 |
| `idempotency_key` | `text` | 可空 | `POST /sessions` 重试键 |
| `request_hash` | `text` | 可空 | 同键不同请求冲突检测 |
| `created_at` | `timestamptz` | `now()` | 创建时间 |
| `updated_at` | `timestamptz` | `now()` | 最近变更时间 |

约束与索引：

- 外键索引 `sessions(project_id)`。
- `UNIQUE (codex_thread_id)`，仅对非空 `codex_thread_id` 生效。
- `UNIQUE (idempotency_key)`，仅对非空 `idempotency_key` 生效。v1 为单一 Runtime 所有者，不预留租户字段。
- `idempotency_key` 和 `request_hash` 必须同时为空或同时非空。
- `last_session_sequence >= 0`。

写入者：Run Server、Bootstrap 导入器。读取者：Run Server Runtime API。

## 6. `runtime.runs`

一条 Run 表示一次客户端提交，或 Bootstrap 导入的一条既有 Codex Turn。这样客户端只需要一种列表和详情模型。

| 字段 | 类型 | 空值/默认 | 约束与说明 |
| --- | --- | --- | --- |
| `run_id` | `text` | 非空 | 主键，客户端提交后的凭证 |
| `session_id` | `text` | 非空 | 外键到 `sessions.session_id` |
| `codex_turn_id` | `text` | 可空 | Agent 创建或 Bootstrap 识别后回填 |
| `status` | `text` | 非空 | 见下方状态集合 |
| `state_version` | `bigint` | `1` | 状态条件更新版本 |
| `idempotency_key` | `text` | 可空 | live Run 创建重试键 |
| `request_hash` | `text` | 可空 | 同键不同 Prompt 冲突检测 |
| `prompt_text` | `text` | 可空 | 用户输入；是否保留及加密由阶段 0 冻结 |
| `cancel_requested_at` | `timestamptz` | 可空 | 非空表示已接受取消意图 |
| `persisted_through_sequence` | `bigint` | `0` | PostgreSQL 已连续提交的 Agent 事件水位 |
| `final_sequence` | `bigint` | 可空 | Agent 声明的终态事件序号 |
| `error_code` | `text` | 可空 | 稳定、脱敏错误码 |
| `error_message` | `text` | 可空 | 可展示的脱敏错误说明 |
| `created_at` | `timestamptz` | `now()` | 排队或导入时间 |
| `started_at` | `timestamptz` | 可空 | 实际开始执行时间 |
| `finished_at` | `timestamptz` | 可空 | 进入终态时间 |
| `updated_at` | `timestamptz` | `now()` | 最近状态或结果更新时间 |

状态集合：`queued`、`dispatching`、`accepted`、`running`、`waiting_agent`、`recovering`、`finalizing`、`completed`、`failed`、`canceled`。

约束与索引：

- 索引 `runs(session_id, created_at, run_id)` 支持会话内 Cursor 列表。
- `UNIQUE (session_id, idempotency_key)`，仅对非空 `idempotency_key` 生效。
- `UNIQUE (codex_turn_id)`，仅对非空 `codex_turn_id` 生效。
- `idempotency_key` 和 `request_hash` 必须同时为空或同时非空；Bootstrap 导入记录两者为空。
- `CHECK status IN (...)` 使用本文列出的 10 个状态；`state_version > 0`。
- `persisted_through_sequence >= 0`，`final_sequence` 为空或大于 0。
- 只有 `persisted_through_sequence >= final_sequence` 才能从 `finalizing` 进入 `completed`。
- `completed/failed/canceled` 必须有 `finished_at`；非终态不得有 `finished_at`。

写入者：Run Server、Dispatcher、Runtime Event Persister、Bootstrap 导入器。读取者：Run Server。

## 7. `runtime.session_events`

Session Event 解决“Web 不知道 iPhone 创建了新 Run”的问题，也是 Session SSE 断线续传的权威来源。

| 字段 | 类型 | 空值/默认 | 约束与说明 |
| --- | --- | --- | --- |
| `session_id` | `text` | 非空 | 复合主键、外键 |
| `session_sequence` | `bigint` | 非空 | 复合主键，Session 内连续递增 |
| `event_type` | `text` | 非空 | 版本化 allowlist |
| `schema_version` | `integer` | `1` | Payload Schema 版本 |
| `payload` | `jsonb` | 非空 | SSE 所需的最小事件数据 |
| `created_at` | `timestamptz` | `now()` | 数据库提交时间 |

主键：`(session_id, session_sequence)`，且 `session_sequence > 0`。持久事件类型首期只有 `run.created`、`run.status.changed`、`run.completed`；Agent presence 是无 Cursor 的临时 SSE 事件，不写本表。按主键即可完成 `Last-Event-ID` 后续读，不再增加 `event_id`。

写入者：Run Server、Dispatcher、Runtime Event Persister。读取者：Run Server 的 Session SSE 和补查接口。

## 8. `runtime.run_events`

| 字段 | 类型 | 空值/默认 | 约束与说明 |
| --- | --- | --- | --- |
| `run_id` | `text` | 非空 | 复合主键、外键 |
| `agent_sequence` | `bigint` | 非空 | 复合主键，Run 内连续递增 |
| `event_type` | `text` | 非空 | 版本化 allowlist |
| `schema_version` | `integer` | `1` | Payload Schema 版本 |
| `payload` | `jsonb` | 非空 | 文本、工具状态或结构化结果事件 |
| `occurred_at` | `timestamptz` | 非空 | Agent 观测时间 |
| `persisted_at` | `timestamptz` | `now()` | PostgreSQL 提交时间 |

主键：`(run_id, agent_sequence)`，且 `agent_sequence > 0`，同时承担 Runtime Event Persister 幂等约束。事件类型由版本化协议 allowlist 校验，不在数据库重复维护会随协议演进的 CHECK。无需额外 `event_id` 或 Payload Hash 字段；重复序号不会再次推进状态或生成 Session Event。

写入者：Runtime Event Persister、Bootstrap 导入器。读取者：Run Server。

## 9. `runtime.command_outbox`

| 字段 | 类型 | 空值/默认 | 约束与说明 |
| --- | --- | --- | --- |
| `command_id` | `text` | 非空 | 主键 |
| `command_type` | `text` | 非空 | `run.start`、`run.cancel`、`bootstrap.start` |
| `resource_id` | `text` | 非空 | 对应 `run_id` 或 `sync_id` |
| `schema_version` | `integer` | `1` | Payload Schema 版本 |
| `payload` | `jsonb` | 非空 | 版本化命令内容 |
| `status` | `text` | `pending` | `pending/claimed/delivered/dead` |
| `attempt_count` | `integer` | `0` | 领取尝试次数 |
| `next_attempt_at` | `timestamptz` | `now()` | 下次可领取时间 |
| `claimed_by` | `text` | 可空 | 当前 Dispatcher 实例 |
| `claim_expires_at` | `timestamptz` | 可空 | 租约超时后可重新领取 |
| `delivered_at` | `timestamptz` | 可空 | Agent durable 接收后时间 |
| `last_error_code` | `text` | 可空 | 最近稳定错误码 |
| `created_at` | `timestamptz` | `now()` | 命令创建时间 |

约束与索引：

- `UNIQUE (command_type, resource_id)`，同一资源同类命令只创建一次。
- 领取索引 `(status, next_attempt_at, created_at)`，Dispatcher 使用 `FOR UPDATE SKIP LOCKED` 有界领取。
- `CHECK command_type IN ('run.start','run.cancel','bootstrap.start')`。
- `CHECK status IN ('pending','claimed','delivered','dead')`；`attempt_count >= 0`；`delivered` 必须有 `delivered_at`。

写入者：Run Server。更新者：Dispatcher。读取者：Dispatcher。v1 Dispatcher 只向当前唯一已配置 Agent 投递。

## 10. `runtime.sync_jobs`

| 字段 | 类型 | 空值/默认 | 约束与说明 |
| --- | --- | --- | --- |
| `sync_id` | `text` | 非空 | 主键 |
| `status` | `text` | `queued` | `queued/running/completed/failed` |
| `idempotency_key` | `text` | 非空 | Bootstrap HTTP 重试键 |
| `snapshot_id` | `text` | 可空 | Agent 开始扫描后回填 |
| `last_committed_batch_no` | `bigint` | `-1` | 已 durable 提交的最新批次号 |
| `last_batch_checksum` | `text` | 可空 | 同批次重传冲突检测 |
| `item_count` | `bigint` | `0` | 已提交的 Run 数量，用于进度展示 |
| `total_sessions` | `bigint` | `0` | Agent 在快照清单中冻结的会话总数 |
| `processed_sessions` | `bigint` | `0` | 已读取并 durable 提交的会话数 |
| `archived_sessions` | `bigint` | `0` | 最终安全对账软归档的会话数 |
| `archived_projects` | `bigint` | `0` | 最终安全对账软归档的项目数 |
| `reconciliation_applied` | `boolean` | `false` | 是否使用未续传完整快照执行了删除对账 |
| `error_code` | `text` | 可空 | 稳定错误码 |
| `error_message` | `text` | 可空 | 可展示的脱敏说明 |
| `created_at` | `timestamptz` | `now()` | 创建时间 |
| `updated_at` | `timestamptz` | `now()` | 最近检查点时间 |

约束与索引：

- `UNIQUE (idempotency_key)`。
- 全局最多一个 `queued/running` Job，使用部分唯一索引实现。
- `CHECK status IN ('queued','running','completed','failed')`。
- `last_committed_batch_no >= -1`、`item_count >= 0`。
- 每批严格按 `last_committed_batch_no + 1` 提交；重复当前批次 checksum 相同则返回既有 durable ACK，不同则报协议冲突。
- Codex 读取游标只保存在 Agent `bootstrap_syncs`；服务端只保存已经提交的批次水位。
- 客户端断开不取消 Sync Job；Dispatcher 在 Agent 恢复后继续投递持久 Outbox。
- 持久水位续传会重新读取 Agent 目录，因此只允许幂等导入，不允许删除对账；客户端可在完成后创建一次新 Job 获取完整快照。

写入者：Run Server。更新者：Bootstrap Persister。读取者：Run Server、Bootstrap Persister。

## 11. 数据分类与保留

| 表 | 敏感级别 | 保留依据 |
| --- | --- | --- |
| `projects` | 低；仅稳定 ID 和显示名 | 与 Runtime 数据生命周期一致，具体期限阶段 0 冻结 |
| `sessions` | 中；标题可能包含业务信息 | 产品历史保留策略，具体期限阶段 0 冻结 |
| `runs` | 高；包含用户 Prompt 和可展示错误 | Prompt 是否持久、加密与期限必须由阶段 0 人工确认 |
| `session_events` | 中到高；取决于版本化 Payload | 不短于对应 Session 的可恢复窗口 |
| `run_events` | 高；包含回复、工具输出和 Diff | Response/Tool/Diff 分项期限由阶段 0 人工确认 |
| `command_outbox` | 高；Payload 可能包含 Prompt | delivered/dead 后仅保留故障恢复所需窗口 |
| `sync_jobs` | 低；仅状态、水位和脱敏错误 | 初始化验收和故障排查窗口 |

保留策略落地前不得启用自动删除。不得为了将来分析向任何表增加用户画像、设备指纹或通用元数据字段。

## 12. 必须原子的事务

| 操作 | 同一 PostgreSQL 事务内必须完成 |
| --- | --- |
| 创建 Session | 插入 `sessions`；相同幂等键校验 `request_hash` |
| 创建 Run | 插入 `runs`、递增 Session 序号、插入 `session_events`、插入 `command_outbox` |
| 请求取消 | 首次写 `cancel_requested_at`、插入唯一 `run.cancel` Outbox、追加 Session Event；重复请求返回现状 |
| 持久化 Run 事件 | 插入 `run_events`、推进连续水位、必要时更新 Run/Session 状态并追加 Session Event |
| 提交 Bootstrap 批次 | Upsert 本批 Project/Session、插入缺失 Run/Run Event、推进 `sync_jobs` 检查点；仅安全最终批次软归档缺失 Project/Session 并写归档计数，全部成功后才 durable ACK |

## 13. 明确不建立的表

| 不建立 | 原因 | 当前替代 |
| --- | --- | --- |
| 用户、租户、成员关系 | 当前未提出多租户或共享权限需求 | Runtime v1 单一所有者部署边界 |
| Agent | 当前 Relay 明确只允许一个 `local-mac` Agent | Relay 配置身份 + Redis presence；多 Agent 需求确认后再迁移 |
| Agent Key、Enrollment | 配对、密钥轮换尚未形成确定需求 | 独立安全设计通过 ADR 后再建模 |
| `session_turns`、`turn_items` | 与 Run/Run Event 重复 | `runs`、`run_events` |
| 通用幂等记录 | 只有三类创建操作需要幂等 | 聚合记录自己的 `idempotency_key` |
| Sync Batch 历史 | 首期严格串行且逐批 ACK | `sync_jobs` 最新批次号与 checksum |
| 客户端 Cursor | SSE Cursor 属于各客户端本地恢复状态 | `Last-Event-ID` + Session/Run 序号 |
| Redis 镜像、结果快照表 | 可从权威事件生成，额外表会产生双写 | `run_events`；Run 摘要直接聚合终态事件 |
| Runtime 审计和诊断表 | 属于独立 Admin Platform 边界 | Admin Platform 版本化 ingestion |

> [!important] 后续加表门槛
> 只有出现已经确认的产品需求、现有 7 表无法无歧义承载，并且 ADR 说明读写者、事务、迁移和删除策略后，才能新增表。不得以“以后可能需要”为理由预留。

## 14. 实现验收

- [ ] PostgreSQL migration 只创建本文 7 张 Runtime 业务表。
- [ ] 每张表的列、默认值、外键、唯一约束、CHECK 和索引与本文一致。
- [ ] 并发幂等、Outbox 原子性、序号连续性和终态水位有自动测试。
- [x] Bootstrap 重传、断连续传禁止删除对账、完整快照软归档和恢复可见性有自动测试。
- [ ] 数据库角色、保留周期、Prompt/Result 加密策略在阶段 0 冻结并回写本文。
- [x] migration、Obsidian、OpenAPI 和 Apifox Current 已在 2026-08-14 对齐。
