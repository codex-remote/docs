---
title: ADR-003 MVP 演进兼容边界
date: 2026-08-10
tags:
  - ai-coding-remote
  - adr
  - mvp
  - architecture
aliases:
  - ADR-003
status: accepted
decision_date: 2026-08-10
related:
  - "[[MVP WebSocket 协议]]"
  - "[[系统总体架构]]"
  - "[[发布与版本策略]]"
---

# ADR-003 MVP 演进兼容边界

## 决策

MVP 必须是可扩展的基本盘，但预发布协议不承担向后兼容义务。多项目版本直接升级为 `spec_version: "2.0"`，删除固定目录和 `run.*` 模型，不实现 1.0 转换器、双协议 Handler 或旧命令别名。

预发布的结束不由代码可运行、局域网部署或 tag 单独决定。只有满足 [[发布与版本策略]] 的正式发布判定，并由项目 Release Note 明确设置 `release_status: released`，才开始形成兼容性基线。

## 稳定的是架构边界

未来扩展应复用以下职责边界：

- Relay WebSocket Transport：连接生命周期、帧限制、单写循环。
- Relay Router/Registry：消息方向和连接目标。
- Mac Workspace Catalog：本地项目发现与路径授权。
- Mac Codex Adapter：Codex App Server RPC 与通知转换。
- Mac Turn Controller：单次执行、超时、中断和快照。
- iPhone RelayClient：协议编解码与连接恢复。
- iPhone ViewModel：Project、Thread、Turn 的界面状态。

## 不稳定的是预发布 Wire Contract

- 2.0 是当前三个仓库的唯一协议。
- 不解析 `spec_version: "1.0"`。
- 不接受 `run.start`、`run.cancel` 等旧消息。
- 不保留 `--working-dir`、`run` 命令或对应环境变量。
- 若未来语义发生破坏性变化，再显式提升协议大版本并同步升级三端。

## 未来能力的接入方式

| 新能力 | 新增组件 | 复用的基本盘 |
| --- | --- | --- |
| 用户鉴权 | Relay 握手 Auth Middleware、Principal | WebSocket Transport、Router |
| 设备身份 | Device Registry、配对与密钥 | Agent Relay Client |
| Admin/Diagnostics | 独立 Admin Server、Admin Web、Collector | 三端实时通信链路 |
| 用户/行为/审计 | Admin 模块、PostgreSQL、SLS | 共享关联 ID，不共享 Relay 状态 |
| 可靠业务 Task | 待后续 ADR 确定的 Task Service、Repository、Lease | Dispatcher 最终创建 Turn 请求 |
| 多 Mac | Map-based Connection Registry、目标设备字段 | 每台 Mac 的 Catalog/Adapter/Controller |
| 可靠投递 | Sequence、ACK、Inbox/Outbox | 现有事件产生点和连接循环 |
| 多执行器 | Executor Adapter Registry | Workspace Catalog 与 Relay Transport |

## 防冲突规则

- WebSocket Handler 不调用 Codex，不管理 Turn。
- Relay 不成为本地项目和 Codex Thread 的事实来源。
- SwiftUI View 不直接拼装 JSON。
- 远程 payload 不包含绝对路径、Shell 命令、二进制路径或启动参数。
- 未来 Task 是持久化业务意图，不能直接冒充 Codex Thread 或 Turn。
- Admin、数据库、日志采集、鉴权和设备路由通过各自边界接入，不能侵入 Codex App Server Adapter。
- 用户、行为、技术日志、管理审计和附件不得混入 Relay Wire Contract。

## 结果

系统可以在不重写执行核心的前提下增加数据库、鉴权、任务状态和多 Mac；但这是代码架构可演进，不是旧协议兼容承诺。

当前 Runtime 发布状态见 [[0.2.0 Homebrew 发布候选]]。
