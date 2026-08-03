---
title: 2026-08-02 TFly Portal 订阅格式拆分与公开路径实施验收记录
type: implementation-validation
status: verified
created: 2026-08-02
updated: 2026-08-02
sensitivity: internal
project: TFly Portal
tags:
  - portal
  - subscription
  - clash
  - v2rayng
  - validation
sources:
  - 2026-08-02 本次会话中用户确认的订阅格式、独立轮换与公开路径决策
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\migrations\009_subscription_token_formats.sql
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\store.go
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\server.go
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\integration_test.go
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\web\templates\dashboard.html
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\web\templates\token.html
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\web\templates\clients.html
  - "[[2026-08-02 TFly Portal 订阅模板与统一部署实施验收记录]]"
---
# 2026-08-02 TFly Portal 订阅格式拆分与公开路径实施验收记录

> [!summary] 阶段结论
> Portal 已将 Clash/Mihomo 与通用节点订阅拆分为两个独立 Token。两种格式可以分别生成和轮换，互不影响。用户侧公开地址收敛为 `/clash/{portalToken}` 与 `/{portalToken}`；旧 `/sub/{portalToken}` 不保留兼容并明确返回 `404`。本地单元测试及 PostgreSQL 集成测试通过；当前尚未执行本批生产发布和真实客户端回归。

## 已接受决策

- Clash/Mihomo 使用独立订阅，面向 Clash Verge Rev、Clash Party 等 Mihomo 客户端。
- 通用节点订阅使用独立 Token，面向 v2rayNG、Shadowrocket 等支持标准节点订阅的客户端。
- 电脑与手机上的两种订阅必须能够独立重新生成；轮换一种格式不得使另一种格式失效。
- 继续使用专用订阅域名，避免调整 DNS、证书和部署配置。
- 通用订阅公开地址移除重复的 `/sub/`，使用域名根路径 `/{portalToken}`。
- Clash 公开地址继续使用 `/clash/{portalToken}`。
- 用户确认此前没有正式用户依赖 `/sub/{portalToken}`，因此不提供兼容或重定向；该路径明确返回 `404`。
- Gateway 内部的 `/sub/{3xuiSubscriptionID}` 是 Portal 与 tfly-sub-gateway 之间的服务协议，继续保留，不暴露原生 3x-ui 订阅 ID。

## 实现事实

### Token 数据模型与安全边界

- 迁移 `009_subscription_token_formats.sql` 为 `subscription_tokens` 增加 `format` 字段，只允许 `clash` 和 `sub`。
- 活跃 Token 唯一约束调整为“每个用户、每种格式最多一个有效 Token”。
- Portal 只保存 Token 哈希、前缀和格式；完整地址仍只在创建或轮换时展示。
- Token 解析同时校验请求格式、账号状态、服务绑定和权益状态，Clash Token 与通用 Token 不可串用。
- 拆分前的有效 Token 迁移为 Clash Token；在用户生成独立通用 Token 前保留一次受控过渡，避免迁移瞬间破坏旧状态。拆分完成后两种格式独立轮换。

### 页面与交互

- 用户中心将订阅区改为 Clash/Mihomo 与通用节点订阅两张独立卡片。
- 每张卡片拥有独立的生成或重新生成入口，并明确提示只会使当前格式的旧链接失效。
- 生成结果页分别显示、复制和测试两种完整地址。
- 客户端中心明确说明 Clash 使用 `/clash/`，通用订阅使用域名根路径。
- 页面中的地址默认脱敏；知识库不记录任何真实订阅 Token。

### 公开与内部路由

| 用途 | 路径 | 状态 |
|---|---|---|
| Clash/Mihomo 公开订阅 | `/clash/{portalToken}` | 当前入口 |
| 通用节点公开订阅 | `/{portalToken}` | 当前入口 |
| 旧通用公开订阅 | `/sub/{portalToken}` | 明确返回 `404` |
| Portal 到 Gateway 的 Clash 请求 | `/clash/{3xuiSubscriptionID}` | 内部保留 |
| Portal 到 Gateway 的通用请求 | `/sub/{3xuiSubscriptionID}` | 内部保留 |

## 验证证据

验证环境为 Windows PowerShell、本地项目 `$env:WORKS_ROOT\tfly-portal`、仓库内捆绑 Go 工具链和测试启动的 PostgreSQL 17。

| 验证项 | 结果 |
|---|---|
| `go test ./portal/...` | 通过 |
| PostgreSQL 集成测试 `TestPortalAuthenticationSmokeWithPostgres` | 通过 |
| Clash 与通用 Token 分别生成 | 通过 |
| Clash 与通用 Token 不能串用 | 通过，错误格式返回 `404` |
| 通用根路径代理到 Gateway `/sub/{3xuiSubscriptionID}` | 通过 |
| 旧 `/sub/{portalToken}` | 通过，明确返回 `404` |
| Clash Token 独立轮换 | 通过，通用 Token 继续可用 |
| 通用 Token 独立轮换 | 通过，Clash Token 继续可用 |

关键源码 SHA-256：

| 文件 | SHA-256 |
|---|---|
| `portal/internal/portal/server.go` | `85A3A2936F6EC3D36A961BF47F710E649DB099A81506E58CCE6DB82B0ACF7475` |
| `portal/internal/portal/store.go` | `A6CD6674779FD564272D372F8740B68FBDB4D5E829703B3A00256C24B4A50C78` |
| `portal/internal/portal/integration_test.go` | `A2960BD810EEE7542E816B2D9853F715520E5F835D187B810BDA545A9C2BFCD0` |
| `portal/internal/portal/migrations/009_subscription_token_formats.sql` | `C31BA05332B3ACA3EFB12454BDF5D21D40924C037ED2F51F16C0F328730E567A` |

## 状态边界

- **实现完成**：订阅格式拆分、独立轮换、公开路径收敛、页面说明和测试已完成。
- **本地验收通过**：单元测试和 PostgreSQL 集成测试均已取得通过证据。
- **生产就绪**：代码层面已达到发布条件，但尚无本批生产部署回执。
- **生产已上线**：否，当前不能标记为已上线。
- **仍需人工验收**：发布 Portal 后重新生成通用订阅，在 v2rayNG 验证根路径订阅；在 Clash Verge Rev 或 Clash Party 验证 `/clash/` 订阅，并确认两台设备同时更新不互相影响。
- **安全待办**：会话中曾出现完整订阅 Token；生产发布后必须重新生成对应通用订阅，使旧 Token 失效。笔记未保存该凭据。

## 下一步

1. 使用统一发布脚本选择 Portal 组件并完成生产发布。
2. 检查 `/healthz`、用户中心生成页面、通用根路径和 Clash 路径。
3. 在真实 v2rayNG 与 Mihomo 客户端分别更新订阅并执行节点连接测试。
4. 重新生成曾暴露的通用订阅 Token，并确认旧地址返回 `404`。
5. 取得发布、健康检查和真实客户端证据后，再创建生产发布续验记录。

## 相关笔记

- [[TFly Portal项目索引]]
- [[2026-08-02 TFly Portal 订阅模板与统一部署实施验收记录]]
- [[2026-08-02 TFly Portal 用户中心界面与流量状态续验记录]]
- [[TFly Portal 标准发布与回滚手册]]
