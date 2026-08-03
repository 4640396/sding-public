---
title: 2026-08-02 TFly Portal 客户邀请功能需求与验收记录
type: requirement-implementation-acceptance-record
status: verified
created: 2026-08-02
updated: 2026-08-02
sensitivity: internal
project: TFly Portal
tags:
  - portal
  - invitation
  - onboarding
  - authentication
  - admin
sources:
  - 2026-08-02 本次会话中用户确认的邀请客户需求
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\server.go
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\store.go
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\xui.go
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\migrations\006_user_invitations.sql
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\integration_test.go
  - "[[2026-08-02 TFly Portal 第一版实施与生产发布验收记录]]"
---
# 2026-08-02 TFly Portal 客户邀请功能需求与验收记录

> [!note] 后续密码规则变更
> 本记录保留邀请功能首次验收时“至少 12 位”的历史事实。2026-08-02 后续需求已将当前密码规则调整为 8–128 个字符；当前实现与验证结果见 [[2026-08-02 TFly Portal 账号设置与自助开通实施验收记录]]。

> [!summary] 阶段结论
> 客户邀请功能已实现并通过单元测试、静态检查和真实 PostgreSQL 端到端验收。管理员可从未绑定的启用服务账号创建邀请，客户通过一次性邮件链接设置密码并激活账号；重发和撤销都会使旧链接失效。核心功能及 3x-ui 响应兼容修复已经生产发布，详见续验记录。

## 已接受需求

- 管理员从后台发起邀请，不向客户发送明文初始密码。
- 管理员填写客户真实收件邮箱，并选择尚未绑定的服务账号。
- 上游服务账号标识只关联流量、有效期和订阅数据，不被当作真实邮箱。
- 客户点击一次性链接，自行设置密码后激活账号。
- 已有正常 Portal 账号不重复创建，只增加服务绑定并发送通知。
- 管理员可以重发或撤销待接受邀请。
- 公共注册继续保留，邀请流程作为更顺畅的管理员开通路径。

## 业务流程

```mermaid
flowchart LR
    A[管理员进入邀请客户] --> B[选择未绑定服务账号]
    B --> C[填写客户真实邮箱]
    C --> D[创建账号与服务绑定]
    D --> E[发送一次性邀请]
    E --> F[客户设置密码]
    F --> G[激活并登录]
    G --> H[查看流量、有效期与订阅]
```

## 已实现功能

### 管理端

- 独立“邀请客户”入口。
- 通过上游客户列表接口读取服务账号，只展示启用且未绑定的账号。
- 订阅标识由服务端读取，不信任浏览器提交的隐藏值。
- 用户列表显示“邀请待接受”，并提供重发与撤销。
- 待邀请账号不能通过普通启用按钮绕过密码设置。
- 管理概览显示待接受邀请数量。

### 客户端

- 邀请邮件包含一次性 `/invitation?token=...` 链接。
- 客户需要两次输入一致且至少 12 位的密码。
- Token 过期、撤销、重发替换或已经使用时拒绝激活。
- 成功后账号变为 `active`，邮箱标记为已验证，并保留服务绑定。

### 数据与安全

- 用户状态增加 `pending_invitation`，Token 类型增加 `accept_invitation`。
- `PORTAL_INVITATION_TTL` 默认 `24h`。
- 数据库只保存 Token 摘要，不保存邮件中的明文 Token。
- 重发先消费旧 Token；撤销消费当前 Token。
- Token 消费、密码写入和账号激活在同一事务中完成。
- 创建、重发、撤销和接受邀请均写入审计日志。

## 验收证据

- `go test ./...`：通过。
- `go vet ./...`：通过。
- 真实 PostgreSQL 集成测试：通过，迁移首次和重复执行均通过。
- 已验证已绑定账号从候选列表排除、邀请激活、链接防重放、重发使旧链接失效、撤销使链接失效。
- 已验证上游客户列表的排序、去重、启用优先和缺失订阅标识过滤。
- 审计验证覆盖创建邀请 3 次、接受 2 次、重发 1 次、撤销 1 次。

## 发布工件与状态

- 发布文件：`$env:WORKS_ROOT\tfly-portal\releases\tfly-portal-linux-amd64`。
- SHA-256：`6D0279A49DB68A94AA4571F2470A7B1BDDE93FB9DD5E5CE264E2A18F0F47570F`。
- 自动发布脚本：`$env:WORKS_ROOT\tfly-portal\tools\deploy\publish-tfly-portal.ps1`。
- **实现完成**：是。
- **本地验收通过**：是。
- **生产部署完成**：核心邀请功能与 3x-ui 兼容修复已完成；后续备注展示增强待发布确认。

## 生产发布后待验收

- [x] 后台能读取真实上游未绑定服务账号。
- [ ] Resend 邀请邮件实际送达。
- [ ] 客户完成设置密码和登录。
- [ ] 流量、有效期和订阅信息与服务端一致。
- [ ] 生产重发与撤销行为符合预期。

## 关联记录

- [[TFly Portal项目索引]]
- [[TFly Portal 标准发布与回滚手册]]
- [[2026-08-02 TFly Portal 第一版实施与生产发布验收记录]]
- [[2026-08-02 TFly Portal 邀请功能生产发布续验记录]]
