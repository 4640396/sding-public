---
title: 2026-08-02 TFly Portal 用户中心界面与流量状态续验记录
type: implementation-acceptance-record
status: verified
created: 2026-08-02
updated: 2026-08-02
sensitivity: internal
project: TFly Portal
tags:
  - portal
  - ui
  - dashboard
  - traffic
  - 3x-ui
sources:
  - 2026-08-02 本次会话中用户提供并逐轮确认的用户中心与流量卡片截图
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\server.go
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\web\templates\dashboard.html
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\web\templates\layout.html
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\web\static\app.css
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\ui_preview_test.go
  - $env:WORKS_ROOT\tfly-portal\releases\tfly-portal-linux-amd64
---
# 2026-08-02 TFly Portal 用户中心界面与流量状态续验记录

> [!summary] 阶段结论
> 用户中心首页已完成订阅就绪区、三步服务进度、流量卡片和订阅到期卡片的布局收敛。有限额度与不限流量两种 3x-ui 状态均已实现独立且一致的视觉表达，并通过本地自动化测试、静态检查、浏览器截图和 Linux amd64 构建验收。本批二进制尚未取得生产发布回执，不记为已上线。

## 已确认设计边界

- 用户确认以现有 HTML 页面作为降级后的开发来源，不继续消耗额度维护结构化 Figma 交付。
- 订阅就绪区采用三段式布局：状态说明、复制订阅链接、配置客户端；下载按钮不再横跨过宽区域。
- 页面减少无意义留白，但保留卡片分区、深色视觉语言和桌面端双栏结构。
- 流量展示必须使用 3x-ui 返回的真实数据，不伪造总额度或使用百分比。
- 有限额度和不限流量使用统一的视觉语言，但语义保持独立。

## 实现结果

### 订阅与引导区域

- “订阅已就绪”调整为横向三段流程，复制链接和配置客户端成为清晰的连续步骤。
- 服务启用进度继续展示“服务已开通 → 订阅已生成 → 导入客户端”，并保留前往客户端中心入口。
- 用户中心状态、订阅 Token、客户端配置和最后在线信息继续由原有服务端数据驱动。

### 流量数据链路

- `server.go` 在 3x-ui 状态可用时始终记录已用流量。
- `TotalBytes > 0` 时计算并限制 `UsagePercent` 到 `0–100`，同时展示已用、剩余和总量。
- `TotalBytes == 0` 时按 3x-ui 语义视为不限流量；不把它硬编码成 100 GB 或其他虚构套餐。
- 数据源不可用与服务尚未开通继续使用独立空状态，不与不限流量混淆。

### 有限额度视觉状态

- 采用带边框的额度轨道、真实渐变填充和居中百分比徽标。
- 0% 状态仍显示完整轨道与 `0%` 徽标，但不绘制虚假已用进度。
- 下方三列显示已用、剩余和总量，并补充当前用量占总额度说明。
- 本地验收状态使用 `0 GB / 200 GB / 200 GB / 0%`，对应用户提供的真实页面状态。

### 不限流量视觉状态

- 使用完整渐变轨道和居中的 `∞` 标识，不再以移动色块模拟未知百分比。
- 下方三列改为已用、套餐“不限流量”和状态“持续可用”，消除“剩余不限 / 共不限”的重复信息。
- 补充“无流量上限，无需关注剩余额度”说明，并保留轻微流光；系统减少动态效果时自动降级。

## 验收证据

- `go test ./...`：通过。
- `go vet ./...`：通过。
- UI 预览生成测试：通过。
- 本地浏览器桌面端截图验收：有限额度与不限流量状态均无横向溢出。
- Linux amd64 发布包构建：通过。
- 发布包 SHA-256：`B6A14B6F8061D0434D0548B5E5E4B711D11CDE7869562C830200EB99720F8D27`。

### 关键源码哈希

| 文件 | SHA-256 |
|---|---|
| `portal/internal/portal/web/templates/dashboard.html` | `BAA110B6893100CC828DB515273B900042DB17FE874B1CE5838D0B4D53FA32AE` |
| `portal/internal/portal/web/static/app.css` | `B9A12C2BE0FE2C4916459679F2F9DE988C0669FFA51CB588A41470A32E550773` |
| `portal/internal/portal/server.go` | `EDBB9D1CD466EAE57C32A30CAD69804AC7536D9858E0E40AE9C165D06ADA1842` |

## 状态边界

- **实现完成**：是。
- **本地自动化验收通过**：是。
- **本地桌面端视觉验收通过**：是。
- **生产就绪**：发布包已生成，但尚未执行本批生产发布和发布后业务验收。
- **生产已上线**：否，当前无新版本发布成功回执。
- **未验证项**：生产端有限额度与不限流量账号切换、移动端真实设备视觉、发布后浏览器缓存更新。

## 下一步

1. 按 [[TFly Portal 标准发布与回滚手册]] 发布当前 Linux amd64 二进制。
2. 分别使用 `TotalBytes > 0` 与 `TotalBytes == 0` 的生产测试账号复核流量卡片。
3. 检查桌面端、移动端、浏览器缓存和 `prefers-reduced-motion` 降级状态。
4. 取得生产发布、健康检查和业务页面截图后，再创建生产发布续验记录。

## 关联记录

- [[TFly Portal项目索引]]
- [[2026-08-02 TFly Portal 第一版实施与生产发布验收记录]]
- [[2026-08-02 TFly Portal 账号设置与自助开通实施验收记录]]
- [[TFly Portal 标准发布与回滚手册]]
