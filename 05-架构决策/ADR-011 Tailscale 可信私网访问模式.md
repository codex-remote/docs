---
title: ADR-011 Tailscale 可信私网访问模式
date: 2026-10-10
tags:
  - ai-coding-remote
  - adr
  - tailscale
  - runtime
status: accepted
implementation_status: implemented-source
decision_date: 2026-10-10
related:
  - "[[Mobile Web Gateway 与 Runtime 鉴权架构]]"
  - "[[ADR-008 Mobile Web Gateway 作为统一用户入口]]"
  - "[[ADR-010 TryCloudflare Quick Tunnel 作为临时公网轮询入口]]"
---

# ADR-011 Tailscale 可信私网访问模式

## 背景

个人用户需要在蜂窝网络或外部 Wi-Fi 下连接家里或办公室 Mac 上的
Codex，同时保持 Runtime、项目、凭据和数据库位于 Mac。公开端口转发、随机
Tunnel 和尚未完成硬化的公网 Gateway 都不满足当前安全与可靠性边界。

## 决策

1. Runtime 提供 `lan` 和 `tailscale` 两种可选访问模式，现有安装默认保持
   `lan`。
2. 两种模式复用同一个 Mobile Web Gateway、Runtime Auth 和内部 Loopback
   拓扑，不迁移 PostgreSQL、Valkey、Mac Agent 或项目数据。
3. `tailscale` 模式只使用已连接的官方 Tailscale 客户端信息；默认使用稳定的
   MagicDNS 短名称，并提供 Tailnet IPv4 回退。不要求家庭固定公网 IP、路由器
   端口转发或云端 Relay。
4. CLI 在切换模式、配对和诊断时读取 Tailscale 状态。找不到客户端、未登录、
   未连接或没有 Tailnet IPv4 时拒绝启用该模式。
5. 模式决定 Runtime 对外展示和配对使用的 Origin。Origin 改变后必须重新配对。
6. 配对前检查同一 Tailnet 内的在线 iOS 设备；一台时自动选择，多台时由交互
   终端选择，自动化环境必须显式指定。该选择只确认目标并命名客户端，不替代
   Runtime Auth，也不把 Pairing Grant 与 Tailscale 设备密钥绑定。
7. Gateway 仍保留 LAN 恢复路径；Tailnet 内的访问范围由 Tailscale ACL 或
   Grants 限制到 Owner 设备和 Gateway 端口。
8. 不启用 Tailscale Funnel，不把 Gateway 发布到普通互联网，也不把 DERP
   当作业务信任边界。Runtime Auth 继续执行最终认证和 Scope 授权。

## 客户端兼容

Mobile Web 通过 MagicDNS 短名称或 Tailnet IPv4 使用现有 HTTP、SSE 或
JSON Poll 契约。原生
iPhone App 使用相同 Runtime URL、Pairing Grant 和 Keychain Token。App 保留
`NSAllowsLocalNetworking`，不增加 `NSAllowsArbitraryLoads`；当前 Apple 平台允许
该声明覆盖 IPv4、IPv6、无限定域名与 `.local` 地址的本地资源加载。

## 运维与验收

- `codex-remote network lan|tailscale` 负责选择模式。
- `status` 同时报告活动、LAN、Tailnet 和 MagicDNS 地址。
- `doctor` 在 Tailscale 模式下验证客户端状态与 Tailnet Gateway 健康。
- 真机验收必须关闭 iPhone Wi-Fi，通过蜂窝网络完成配对、刷新、Run 提交、
  SSE 或 Poll 恢复、锁屏恢复和撤销。
- Mac 必须保持开机、联网并允许 Codex Remote 与 Tailscale 后台运行。

## 不在本决策范围

- 稳定公网 HTTPS Origin、Funnel 或路由器公网端口映射。
- Headscale、自建 DERP 或 Peer Relay。
- 远程唤醒处于关机或深度睡眠状态的 Mac。
