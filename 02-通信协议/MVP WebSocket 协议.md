---
title: MVP WebSocket 协议
date: 2026-08-10
tags:
  - ai-coding-remote
  - mvp
  - websocket
  - protocol
aliases:
  - MVP Protocol
status: active
related:
  - "[[端到端任务时序图]]"
  - "[[会话历史读取]]"
---

# MVP WebSocket 协议

## 版本

当前唯一协议版本是 `2.0`。Relay、Mac Agent 和 iPhone 对其他版本直接拒绝，不兼容旧 `1.0 run.*` 消息。

```json
{
  "spec_version": "2.0",
  "message_id": "8b8b1d58f36a4b9f81f44b29351cb1aa",
  "type": "turn.output",
  "occurred_at": "2026-08-10T02:20:14.481Z",
  "trace_id": "78ec90c815994f8e9017ce7259d6444e",
  "sender": { "kind": "device", "id": "local-mac" },
  "payload": {}
}
```

Envelope 字段全部必填；`occurred_at` 使用 RFC 3339；`trace_id` 关联一次请求和其响应事件。Relay 根据连接端点覆盖 `sender`，但当前不把它当作身份凭据。

## 消息矩阵

| 类型 | 方向 | 作用 |
| --- | --- | --- |
| `agent.hello` | Agent → App | Agent 名称、版本和初始状态 |
| `agent.status` | Agent/Relay → App | `idle`、`running`、`offline` 与当前执行 ID |
| `project.list` | App → Agent | 请求本地项目列表 |
| `project.snapshot` | Agent → App | 返回允许的 Git 项目 |
| `thread.list` | App → Agent | 请求某项目的 Codex 会话 |
| `thread.snapshot` | Agent → App | 返回该项目的会话 |
| `thread.read` | App → Agent | 读取指定会话的持久化 Turns 和 Items |
| `thread.detail` | Agent → App | 返回结构化会话历史 |
| `turn.start` | App → Agent | 在新 Thread 或已有 Thread 中发送 prompt |
| `turn.started` | Agent → App | Codex 已创建 Thread/Turn |
| `turn.output` | Agent → App | assistant/stdout/stderr 流式输出 |
| `turn.snapshot` | Agent → App | 重连恢复当前执行 |
| `turn.interrupt` | App → Agent | 中断当前 Turn |
| `turn.interrupted` | Agent → App | Turn 已中断 |
| `turn.completed` | Agent → App | 成功结果、修改文件和 Diff |
| `turn.failed` | Agent → App | 执行失败 |
| `turn.rejected` | Relay/Agent → App | Agent 离线、忙碌或请求非法 |

## 项目发现

请求：

```json
{
  "spec_version": "2.0",
  "message_id": "e3491ab74ac044969f030f46505bc67b",
  "type": "project.list",
  "occurred_at": "2026-08-10T02:20:00Z",
  "trace_id": "52a3effa6c164230950523b546e27dd2",
  "sender": { "kind": "user", "id": "local-user" },
  "payload": {}
}
```

响应的每个项目包含 `id`、`name`、`path`、`thread_count` 和可选 `updated_at`。`path` 仅用于用户辨认；后续请求必须发送 `project_id`，不能回传或替换为路径。

## 会话查询

```json
{
  "spec_version": "2.0",
  "message_id": "bab6830f121e43128c5f19ee4104e113",
  "type": "thread.list",
  "occurred_at": "2026-08-10T02:20:02Z",
  "trace_id": "e12b630c00f04de5b08419159f682eb4",
  "sender": { "kind": "user", "id": "local-user" },
  "payload": { "project_id": "project_233eb15234dbff1d" }
}
```

`thread.snapshot` 返回 `project_id` 和 `threads[]`。Thread 字段包含 `id`、`project_id`、`title`、`preview`、`status`、`source`、`updated_at`。

## 会话历史

App 使用 `project_id` 和 `thread_id` 发送 `thread.read`；Mac Agent 验证归属后调用稳定的 Codex `thread/read(includeTurns: true)`，再返回 `thread.detail`。响应保留 Turns 和 Items 的顺序，并为消息、命令、文件变更等常见 Item 提供稳定字段。

历史 payload 目标上限为 200 KiB；超限时保留最新历史并设置分层 `truncated`。完整请求、响应、字段表和调试命令见 [[会话历史读取]]。

## 开始 Turn

新会话省略 `thread_id`：

```json
{
  "spec_version": "2.0",
  "message_id": "816ab585c3114b5aa9476e483210378c",
  "type": "turn.start",
  "occurred_at": "2026-08-10T02:20:05Z",
  "trace_id": "4ba9f13374334219a22457606517a550",
  "sender": { "kind": "user", "id": "local-user" },
  "payload": {
    "project_id": "project_233eb15234dbff1d",
    "prompt": "检查当前修改并运行相关测试。"
  }
}
```

继续会话时增加 `thread_id`。Mac Agent 必须验证该 Thread 的 `cwd` 属于 Project，不能跨项目恢复会话。

## 输出与完成

`turn.output.payload` 包含 `project_id`、`thread_id`、`turn_id`、`stream` 和 `text`。`stream` 取 `assistant`、`stdout` 或 `stderr`。

`turn.completed.payload` 包含：

- `project_id`、`thread_id`、`turn_id`
- `duration_ms`
- `summary`
- `changed_files[]`
- `diff`
- `diff_truncated`

Diff 超过限制时截断并标记，不引入对象存储。

## 拒绝与错误

`turn.rejected.payload` 只有 `code` 和 `message`。MVP 主要错误码：

- `AGENT_OFFLINE`
- `AGENT_BUSY`
- `PROJECT_NOT_FOUND`
- `THREAD_NOT_FOUND`
- `TURN_NOT_FOUND`
- `MESSAGE_INVALID`
- `CODEX_START_FAILED`
- `TURN_TIMEOUT`
- `INTERNAL_ERROR`

## 连接规则

- `/ws/app` 与 `/ws/agent` 区分连接角色，不做应用层认证。
- App 只能发送 `project.list`、`thread.list`、`thread.read`、`turn.start`、`turn.interrupt`。
- Agent 只能发送状态、快照、`thread.detail` 和 Turn 事件。
- Relay 每个连接只有一个写循环，并使用有界队列和帧上限。
- Relay 不缓存项目、会话、prompt、输出或 Diff。
- Agent 离线时 Relay 立即返回 `turn.rejected/AGENT_OFFLINE`。
- Agent 忙碌时不排队，立即返回 `AGENT_BUSY`。
