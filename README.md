---
title: Codex Remote 文档仓库
tags:
  - codex-remote
  - index
aliases:
  - README
status: active
updated: 2026-08-23
---

# Codex Remote 文档仓库

这是 Codex Remote 的产品、架构、协议和发布知识库。应用代码保持五个独立仓库：`iphone-app`、`mobile-web`、`relay-server`、`mac-agent` 和 `admin-platform`；Homebrew 分发另由 `runtime-distribution` 与 `homebrew-tap` 两个独立仓库维护。

> [!success] 当前可用能力
> Apple Silicon 本机已经可以启动完整 Runtime、生成一次性二维码，并在 iPhone Safari 单标签中完成配对、刷新恢复、真实会话、SSE、退出与 Mac 端撤销。详细真机证据以 `mobile-web/docs/iphone-safari-acceptance.md` 为准。

> [!warning] 尚未公开发布
> Runtime `0.2.0` 已通过本机 Homebrew 验收，但 GitHub 组织与远端仓库、不可变 Tag/Release、Developer ID 签名、Apple 公证、第三方许可证清单和干净 Mac 升级/回滚验收尚未完成。当前不能向其他用户宣称已经可以从远端安装。

## 阅读入口

- [[Codex Remote - 项目首页]]：当前状态、仓库边界和下一步
- [[系统总体架构]]：当前组件、数据流和部署边界
- [[Mobile Web Gateway 与 Runtime 鉴权架构]]：局域网入口、配对与鉴权
- [[Run Server HTTPS 与 SSE 接口规范]]：用户侧 Runtime 契约
- [[0.2.0 Homebrew 发布候选]]：本地验收证据和公开发布阻塞项
- [[发布与版本策略]]：Runtime、iPhone 与 Admin 的独立发布规则

## 事实源

| 内容 | 唯一事实源 |
| --- | --- |
| Runtime 发布状态与组件 commit | [[0.2.0 Homebrew 发布候选]] 和 Runtime manifest |
| HTTP/SSE 接口 | `relay-server/apifox/openapi.json` 与 [[Run Server HTTPS 与 SSE 接口规范]] |
| Agent WebSocket | `relay-server/protocol` 与 [[MVP WebSocket 协议]] |
| 真机 Mobile Web 验收 | `mobile-web/docs/iphone-safari-acceptance.md` |
| 安装、修复与卸载 | `runtime-distribution/README.md` 和 `docs/installation-troubleshooting.md` |
| 架构取舍 | `05-架构决策/` 下的 ADR |

已完成的短期开发计划不继续承担状态汇总职责；完成证据进入发布记录、仓库 Changelog 或维护手册。历史 MVP 只保留一份范围定义，避免旧拓扑与当前 Runtime 并列成为实现入口。
