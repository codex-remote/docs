---
title: MVP 版本定义
date: 2026-08-10
tags:
  - ai-coding-remote
  - mvp
  - product
aliases:
  - MVP Scope
status: historical
related:
  - "[[MVP WebSocket 协议]]"
  - "[[系统总体架构]]"
---

# MVP 版本定义

> [!note] 历史文档
> 本文只描述最初 iPhone WebSocket MVP。当前 Mobile Web 已使用 Gateway、Runtime PostgreSQL/Valkey、HTTP/SSE 与应用层鉴权，现行边界见 [[系统总体架构]]。

> [!abstract] 一句话目标
> 在 iPhone 选择 Mac 上的任意已发现 Git 项目，创建或继续一个 Codex 会话，发送开发指令，并实时查看执行输出与结果。

## 产品闭环

MVP 必须完成以下闭环：

1. iPhone 能看到 Mac Agent 在线、空闲或执行中。
2. iPhone 能列出 Mac 配置的工作区根目录下的多个 Git 项目。
3. 选择项目后能查看属于该项目的 Codex 历史会话。
4. 用户能创建新会话或继续已有会话，并发送一个 Turn。
5. iPhone 能实时看到 Codex 输出、停止当前 Turn，并查看 Git Diff 摘要。

## 运行模型

| 项目 | MVP 决策 |
| --- | --- |
| 用户 | 单用户，不做登录与应用层鉴权 |
| Mac | 单 Mac Agent |
| 项目 | 多项目；从一个或多个本地工作区根目录发现 Git 仓库 |
| 会话 | 多个 Codex Thread；由 Codex App Server 持久化和读取 |
| 执行 | 同时只允许一个 Turn |
| Relay | 单实例、纯内存、透明转发 |
| 通信 | iPhone 和 Mac 都主动连接 Relay WebSocket |
| 历史 | Relay 不保存历史；会话历史继续归 Codex 所有 |
| 重连 | Agent 重发状态与当前 Turn 的最近内存日志 |

`Project` 是本地 Git 工作区，`Thread` 是 Codex 持久会话，`Turn` 是会话中的一次用户指令。MVP 不引入业务 Task、任务队列、租约或数据库状态机。

## 本期包含

- SwiftUI 单页控制台与 Relay URL 设置。
- 项目列表、会话列表、新建会话和继续会话。
- Codex App Server 适配，而不是为每次请求直接拼装临时 CLI 命令。
- 实时 assistant、stdout、stderr 输出。
- 完成、失败、中断结果和 Git Diff 摘要。
- Relay 与 Mac Agent 自动重连。
- Relay Docker 镜像和局域网部署方式。

## 明确不包含

- 用户登录、Token、设备配对、RBAC。
- PostgreSQL、Redis、业务 Task CRUD、队列和执行历史。
- 多 Mac 路由、离线任务和可靠消息重放。
- 多执行器、MCP、Claude Code 和本地模型。
- 自动提交、推送、创建 PR 或部署。

> [!warning] 部署边界
> 当前 Relay 没有鉴权，必须只运行在可信局域网、Tailscale 或等价私有网络中，不能直接暴露到公共互联网。

## 基本盘约束

- iPhone 只能提交 `project_id`、可选 `thread_id` 和 prompt，不能提交绝对路径、Shell 命令或 Codex 参数。
- `project_id` 必须由 Mac Agent 的 Workspace Catalog 生成；任意路径输入会被拒绝。
- Mac Agent 通过 Codex App Server 的 `thread/list`、`thread/start`、`thread/resume` 和 `turn/start` 操作本地 Codex。
- Relay 不理解 Codex，不保存 prompt、源码、Diff 或会话历史。
- Transport、Inventory、Turn Controller、Codex Adapter 保持解耦；Relay 实时鉴权接入握手边界，Admin、诊断、用户和行为数据库按独立平台边界接入。

## 协议基线

> [!important] 不提供旧协议兼容
> 多项目版本使用 `spec_version: "2.0"`。旧的 `1.0 run.*`、固定 `working-dir` 和 `run` 命令已移除，不设置兼容层、双写或自动转换。

这里的“可扩展”指代码边界允许未来增加数据库、鉴权、Task 和多 Mac，并不承诺当前预发布协议永久向后兼容。进入稳定发布前，协议可以通过显式大版本继续演进。

## MVP 成功标准

在可信局域网内，开发者可在 10 分钟内启动 Relay 与 Mac Agent；iPhone 能看到至少三个本地 Git 项目，选择其中一个新建或继续 Codex 会话，发送指令、查看流式输出、停止执行并看到最终结果。
