---
title: 2026-07-31 TFly Portal 项目重命名实施记录
type: implementation-record
status: accepted
created: 2026-07-31
updated: 2026-07-31
project: TFly Portal
sensitivity: internal
tags:
  - rename
  - portal
  - go
---
# 2026-07-31 TFly Portal 项目重命名实施记录

## 结论

项目正式命名为 **TFly Portal**，源码根目录迁移到 `$env:WORKS_ROOT\tfly-portal`。仓库采用 Go workspace 组织 `portal/` 与 `gateway/` 两个独立 module；Portal 的 HTML 模板和静态资源通过 `embed` 编译进同一 Go 二进制，不需要单独部署前端。

## 运行边界

- Portal 新二进制与 systemd 模板统一使用 `tfly-portal`。
- 已上线 Gateway 的服务名、环境文件和 `/opt` 安装位置继续使用 `tfly-sub-gateway`，避免仓库改名影响生产运行。
- 历史实施记录中的旧绝对路径和旧组件名保留，作为当时事实证据，不做追溯改写。

## 验收

- `go test ./portal/...`：通过。
- `go test ./gateway/...`：通过。
- 源码、知识库项目目录与 Graphify 本地缓存键完成一致性迁移。

## 导航

- [[TFly Portal项目索引]]