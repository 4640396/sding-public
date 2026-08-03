---
title: 2026-08-02 TFly Portal 账号设置与自助开通实施验收记录
type: requirement-implementation-acceptance-record
status: verified
created: 2026-08-02
updated: 2026-08-02
sensitivity: internal
project: TFly Portal
tags:
  - portal
  - account-settings
  - onboarding
  - admin
  - ui
sources:
  - 2026-08-02 本次会话中用户确认的账号设置、自助申请、后台管理和界面优化需求
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\server.go
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\store.go
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\migrations\007_account_settings.sql
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\migrations\008_onboarding_admin.sql
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\integration_test.go
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\web\templates
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\web\static
---
# 2026-08-02 TFly Portal 账号设置与自助开通实施验收记录

> [!summary] 阶段结论
> 账号设置、自助开通申请、邀请检索、用户筛选分页、邀请投递状态和审计日志功能已完成源码实现。核心业务流程通过普通测试、`go vet`、Linux amd64 构建和真实临时 PostgreSQL 端到端测试；最后追加的账户操作工具条与设置 Tab 界面通过普通测试和 `go vet`。本批改动尚未取得生产发布及真实 SMTP 人工验收回执，不记为已上线。

## 已接受产品边界

- 密码最小长度由 12 个字符调整为 8 个字符，上限保持 128 个字符；不限制为纯数字或恰好 8 位。
- 公共注册继续开放；注册用户验证邮箱后可以主动提交服务开通申请，但不能自行认领 3x-ui 服务账号。
- 管理员仍负责把 Portal 用户与未绑定的 3x-ui 服务账号关联，避免服务冒领。
- 用户可在账号设置中修改密码、验证后修改登录邮箱，并撤销其他设备会话。
- 邮箱变更不修改 3x-ui 客户标识；服务绑定按 `user_id` 保持不变。
- 生产部署继续使用既有自动发布脚本，本阶段没有改变 Gateway 的独立部署边界。

## 已实现功能

### 账号设置

- 新增 `/settings`。
- 修改密码要求输入当前密码、两次一致的新密码；成功后撤销其他设备会话并记录审计。
- 修改邮箱要求当前密码，并向新邮箱发送一次性验证链接；验证成功后才切换登录邮箱。
- 邮箱切换后撤销全部现有会话，要求使用新邮箱重新登录，并向旧邮箱发送安全通知。
- 新增“退出其他设备”，保留当前浏览器会话。
- 新增 `email_change_requests`，保存邮箱变更目标、Token 摘要、有效期和消费状态，不保存明文 Token。

### 用户自助开通与管理员审批

```mermaid
flowchart LR
    A[用户注册并验证邮箱] --> B[用户提交开通申请]
    B --> C[管理员查看待审核申请]
    C --> D{审批结果}
    D -->|批准| E[选择未绑定服务账号]
    E --> F[绑定服务并发送通知]
    D -->|拒绝| G[记录结果并通知用户]
```

- 未绑定用户可从用户中心提交可选说明。
- 同一用户同一时间只能有一条待审核申请。
- 管理员在 `/admin/applications` 批准或拒绝申请。
- 批准时服务端重新读取可用服务账号，并使用服务端取得的订阅标识完成绑定。
- 新增 `service_applications`，记录申请、审批人、审批时间和状态。

### 邀请与用户管理

- 邀请页支持按备注或服务账号标识搜索未绑定账号，并显示匹配数量。
- 输入客户邮箱后可提示账号是否已存在；已有激活账号会直接绑定，无需重新设置密码。
- 用户管理支持邮箱或服务账号检索、账号状态筛选、绑定状态筛选和每页 50 条分页。
- 邀请记录显示邮件发送成功、失败、发送时间和过期状态。

### 审计与界面

- 新增 `/admin/audit`，可按操作者邮箱、目标 ID 和操作类型筛选最近审计记录。
- 用户首页右上角“账号设置”和“退出登录”改为同一行的轻量账户工具条。
- 设置页改为 Tab 布局：桌面端左侧分类、移动端顶部标签，单次只显示一个操作面板。
- Tab 支持鼠标和键盘方向键切换；表单错误保持在对应标签。
- 静态资源版本更新到 `20260802-6`，用于规避浏览器旧 CSS/JavaScript 缓存。

## 数据库迁移

- `007_account_settings.sql`：新增邮箱变更请求表及待验证邮箱唯一约束。
- `008_onboarding_admin.sql`：新增邀请投递状态字段和服务开通申请表。
- Portal 启动时按文件顺序执行幂等迁移；无需手工执行 SQL。

## 验收证据

- `go test ./...`：通过。
- `go vet ./...`：通过。
- Linux amd64 构建：核心功能完成后通过。
- `TestPortalAuthenticationSmokeWithPostgres`：在真实临时 PostgreSQL 17 上通过。
- 集成路径覆盖迁移首次及重复执行、密码修改、其他会话撤销、邮箱变更、防重放、绑定保持、用户申请、管理员批准、邀请投递状态、用户筛选和审计查询。
- 最后追加的账户工具条与设置 Tab 模板通过普通测试和 `go vet`；尚未单独重跑生产发布脚本。

## 当前边界与待验收

- **实现完成**：是。
- **本地核心业务验收通过**：是。
- **最终界面回归通过**：普通测试与 `go vet` 通过，仍需浏览器人工检查桌面端和移动端视觉效果。
- **生产部署完成**：否，当前没有本批版本的发布成功回执。
- **真实 SMTP**：尚未验证新邀请、邮箱修改、密码修改和申请审批通知的实际送达。
- **生产数据库迁移**：尚未执行；随下一次 Portal 发布启动时执行。

## 发布后人工验收清单

1. 普通用户登录后确认账户工具条同一行显示。
2. 检查设置页三个 Tab 的桌面端、移动端和键盘切换。
3. 用测试账号修改密码并验证其他会话撤销。
4. 用可收信的测试邮箱完成邮箱变更，确认绑定和订阅不丢失。
5. 用户提交开通申请，管理员批准并绑定服务账号。
6. 邀请页按备注和账号标识搜索，验证已有账号提示和邮件状态。
7. 检查用户筛选、分页和审计日志。
8. 确认真实邮件送达、本机 `/healthz` 和公网 `/healthz`。

## 关联记录

- [[TFly Portal项目索引]]
- [[TFly Portal 标准发布与回滚手册]]
- [[2026-08-02 TFly Portal 客户邀请功能需求与验收记录]]
- [[2026-08-02 TFly Portal 邀请功能生产发布续验记录]]

