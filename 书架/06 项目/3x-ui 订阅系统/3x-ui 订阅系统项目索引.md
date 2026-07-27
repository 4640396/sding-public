---
title: 3x-ui 订阅系统项目索引
type: project-index
status: draft
created: 2026-07-15
updated: 2026-07-25
sensitivity: internal
tags:
  - 3x-ui
  - subscription
  - Clash
  - Mihomo
---
# 3x-ui 订阅系统项目索引

## 项目目标

在不改变3x-ui原有通用订阅的前提下，为 Clash/Mihomo 用户提供带策略组和分流规则的独立订阅地址，并通过自定义订阅页面完成复制和客户端导入。

## 当前组成

- `3x-ui-sub-gateway`：Go实现的订阅转换网关，读取3x-ui原生 Clash YAML，保留节点字段并生成策略组与规则。
- `3x-ui-sub-theme`：3x-ui自定义订阅落地页，展示流量、有效期和订阅入口。
- Nginx：为转换网关提供公网 HTTPS 入口，Go服务仅监听本机回环地址。

## 代码位置

```text
$env:WORKS_ROOT\3x-ui-sub\3x-ui-sub-gateway
$env:WORKS_ROOT\3x-ui-sub\3x-ui-sub-theme
$env:WORKS_ROOT\3x-ui-sub\releases
```

知识库只保存架构、部署结论和排障记录，不保存构建缓存、二进制源码副本或真实订阅凭据。

## 笔记导航

### 01 需求与决策

- [[3x-ui-sub 用户中心需求]]
- [[2026-07-16 3x-ui-sub 用户中心与支付决策记录]]
- [[2026-07-16 3x-ui-sub 用户中心与支付改造分析]]（设计草案）

### 02 架构与实现

- [[2026-07-16 3x-ui-sub 当前代码事实核验]]

### 03 实施与验收

- [[2026-07-16 3x-ui-sub 用户中心第一阶段实施记录]]
- [[2026-07-17 3x-ui-sub 用户中心第一阶段续验记录]]
- [[2026-07-17 3x-ui-sub 用户中心界面修复与预览收尾记录]]

### 04 运维与迁移

- [[2026-07-15 订阅转换网关部署记录]]
- [[3x-ui 部署、安全与排障]]

### 05 历史与复盘

- [[2026-07-15 3x-ui 订阅系统阶段总结]]
- [[2026-07-16 3x-ui-sub 用户中心与支付待决策清单]]（已由正式决策记录取代）

## 当前状态

- Go网关本机健康检查已通过。
- Nginx HTTPS监听和公开证书校验已通过。
- 通过本机指定解析访问公开域名时，网关返回 `HTTP 200` 和 YAML。
- 公共DNS已能将订阅子域名解析到源站。
- 完整 Mihomo 配置已能通过转换地址交付 Clash Party、Mihomo Party 和 Clash Verge Rev。
- 当前继续保留3x-ui订阅主题作为过渡入口；独立用户中心列入后续开发。
- 仍需轮换曾出现在截图或聊天中的订阅ID。
