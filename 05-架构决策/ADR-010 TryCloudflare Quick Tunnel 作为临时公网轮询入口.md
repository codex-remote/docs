---
title: ADR-010 TryCloudflare Quick Tunnel 作为临时公网轮询入口
date: 2026-08-26
updated: 2026-08-26
tags:
  - ai-coding-remote
  - adr
  - cloudflare
  - polling
aliases:
  - ADR-010
status: rejected
implementation_status: removed
decision_date: 2026-08-26
related:
  - "[[Runtime JSON 轮询传输架构]]"
  - "[[Mobile Web Gateway 与 Runtime 鉴权架构]]"
---

# ADR-010 TryCloudflare Quick Tunnel 作为临时公网轮询入口

## 结论

不采用 TryCloudflare 随机域名作为 Codex Remote 公网入口，相关启动模式和脚本已移除。

## 原因

- Cloudflare 官方仅将 Quick Tunnel 定位为测试与开发能力，不提供 SLA。
- 真实验证出现 API 和 Connector 正常，但随机域名未发布 DNS 的情况；有限重试无法保证可用地址。
- 随机 Origin 会在重启后变化，浏览器需要重新配对，无法满足主要远程入口的稳定性要求。

## 影响

- 局域网 SSE 与 JSON Poll 两种入口继续保留。
- JSON Poll 传输保持边缘无关，不绑定具体 Tunnel 厂商。
- 稳定公网入口需要固定 Origin，并在后续独立架构决策中评估。

官方依据：[Cloudflare Quick Tunnels](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/trycloudflare/)。
