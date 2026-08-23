---
title: ADR-008 Mobile Web Gateway 作为统一用户入口
date: 2026-08-22
tags:
  - ai-coding-remote
  - adr
  - gateway
  - mobile-web
  - security
aliases:
  - ADR-008
status: accepted
implementation_status: implemented
decision_date: 2026-08-22
related:
  - "[[Mobile Web Gateway 与 Runtime 鉴权架构]]"
  - "[[0.2.0 Homebrew 发布候选]]"
  - "[[可靠多端会话与运行时控制面升级方案]]"
  - "[[ADR-004 三仓库独立开发]]"
  - "[[ADR-007 PostgreSQL 权威运行时与 Redis 实时加速]]"
---

# ADR-008 Mobile Web Gateway 作为统一用户入口

## 状态

已接受并实现。固定端口 `18774` 已注册为 `codexremote / mobile-web-gateway / gateway`，启动入口为 `mobile-web/start.sh gateway`。

## 背景

本决策实施前，局域网 Mobile Web 从 Vite `4173/4174` 加载页面，再根据访问主机推导 Run Server `18775`。这种双端口浏览器拓扑依赖宽松 CORS，不能自然承载 HttpOnly Refresh Cookie，也会让未来公网入口同时理解前端与 Runtime 两个服务。

决策时，Runtime v1 已通过 HTTP/SSE 提供用户契约，并约定公网边缘升级 HTTPS；Mac Agent 仍通过本机/主动 WSS 连接。当时需要建立一个局域网可验证、未来可被 Cloudflare 或其他公网入口复用的单一用户入口并接入 Runtime 鉴权；两项现均已完成。

## 决策

1. 在 `mobile-web` 仓库内新增独立 Mobile Web Gateway 服务，不建立新仓库。
2. Gateway 固定监听 `0.0.0.0:18774` 以支持局域网浏览器；Run Server `mobileweb` profile 监听 `127.0.0.1:18775`。
3. 浏览器只访问 Gateway，并通过同源路径消费 `/v1/runtime/*` 与 `/v1/auth/*`。
4. 开发模式下 Gateway 可代理 Vite `4174`；部署模式下直接提供版本化 `dist`，两种模式使用相同 API 路由。
5. Gateway 只代理显式 allowlist，不代理 `/ws/agent`、旧 `/ws/app`、Auth Control、`/status`、调试路径或任意上游。
6. Gateway 不承担业务认证和授权。Run Server 的统一中间件是最终强制边界。
7. Gateway 必须保持 SSE 增量、Cursor、幂等 Header、Cookie 和客户端取消语义。
8. 未来公网连接器只能指向 Gateway；不能直接指向 Run Server、Vite、Mac Agent 或 Auth Control。
9. Gateway 使用 Go 标准库 `net/http` 和 `httputil.ReverseProxy`；SSE 使用立即 Flush，静态资源和精确路径 allowlist 由 Gateway 自身实现。
10. Gateway 是 `mobile-web` 仓库内的独立 Go module 和进程，不导入 `relay-server` 源码；跨仓兼容通过 `run-server-v1` 契约记录、OpenAPI 和测试维护。
11. Runtime Auth 是 Relay 内的独立代码/Schema 模块，通过 Server 组装接口接入；Auth Control 是同进程独立监听器，不宣称为独立微服务。

完整路由、端口、Auth 和故障边界见 [[Mobile Web Gateway 与 Runtime 鉴权架构]]。

## 结果

收益：

- 局域网和公网共享一套前端路径与 Runtime/Auth 契约。
- 浏览器不再依赖 `4174 -> 18775` 跨端口 CORS。
- Refresh Cookie、`X-CodexRemote-Request` CSRF 请求头、SameSite 策略和安全响应头有稳定边界。
- 公网只暴露一个可审计 allowlist 入口，Agent 与本机控制面保持隐藏。
- Gateway、Run Server Auth 和 Runtime Handler 可以分别测试和替换。

代价：

- 新增一个需要监督、健康检查、日志和升级的本地进程。
- Gateway 故障会同时影响页面与 API，但不会取消已持久 Run。
- 部署脚本必须增加精确重启顺序和自重启保护。
- SSE、上游错误、Header 和取消传播必须通过真实代理验证，不能只做静态页面测试。
- 局域网 HTTP 与公网 HTTPS 的 `Secure` Cookie 属性仍有环境差异；自动化测试必须补齐 TLS 行为。

## 被拒绝的方案

### 浏览器继续直接访问 Run Server `18775`

拒绝作为目标入口。它保留跨端口 CORS，扩大公网路径，且无法为 Mobile Web 静态资源和 Runtime API 建立统一 Cookie/Origin 边界。

### 直接把 Vite `4174` 当作 Gateway

拒绝作为部署方案。Vite 是开发工具，不应成为无人值守局域网或公网反向代理、安全头和静态发布边界。

### 将 Mobile Web 静态资源并入 Run Server

拒绝。它会让 `relay-server` 发布物依赖 `mobile-web` 构建产物，破坏五个仓库独立构建、发布和回滚的边界。

### 新建第六个 Gateway 仓库

当前不采用。Gateway 与 Mobile Web 同一用户入口、同一版本化静态发布物和启动文档，拆仓只会增加跨仓发布协调。若未来同时服务多个独立客户端或协议，再以实际需求修订 ADR。

### 只依赖 Cloudflare Access 或 WAF 做鉴权

拒绝。外层入口可以叠加保护，但不能替代 Run Server 的设备 Session、Scope 和资源授权，也不能保护本机绕过 Gateway 的调用。

## 实施约束

- 当前交付与验收状态统一见 [[0.2.0 Homebrew 发布候选]]。
- 实现不得改变 Runtime v1 业务语义或把 SSE 连接生命周期绑定到 Run 生命周期。
- `devrun` 只做全局查找；依赖检查、精确端口清理、健康检查和进程管理留在 `mobile-web` 项目入口。
- 路径拒绝、Header 清理、SPA、SSE 代理与真实 PostgreSQL/Valkey Auth 链路已自动验证；iPhone Safari 单标签闭环已通过，双标签与长时间断线恢复仍是后续门禁。
