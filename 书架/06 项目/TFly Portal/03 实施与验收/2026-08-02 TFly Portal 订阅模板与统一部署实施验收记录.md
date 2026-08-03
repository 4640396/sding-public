---
title: 2026-08-02 TFly Portal 订阅模板与统一部署实施验收记录
type: implementation-validation
status: verified
created: 2026-08-02
updated: 2026-08-02
sensitivity: internal
project: TFly Portal
tags:
  - gateway
  - mihomo
  - deployment
  - validation
sources:
  - $env:WORKS_ROOT\tfly-portal\override.yaml
  - $env:WORKS_ROOT\tfly-portal\gateway\main.go
  - $env:WORKS_ROOT\tfly-portal\gateway\main_test.go
  - $env:WORKS_ROOT\tfly-portal\gateway\install.sh
  - $env:WORKS_ROOT\tfly-portal\gateway\tfly-sub-gateway.env.example
  - $env:WORKS_ROOT\tfly-portal\tools\deploy\publish-tfly.ps1
  - $env:WORKS_ROOT\tfly-portal\README.md
  - "[[TFly Portal 标准发布与回滚手册]]"
---
# 2026-08-02 TFly Portal 订阅模板与统一部署实施验收记录

> [!summary] 阶段结论
> 根目录 `override.yaml` 已成为 tfly-sub-gateway 的唯一完整 Mihomo 配置模板。Gateway 生成订阅时仅注入当前用户的 `proxies`，其余配置与模板保持一致。Portal 与 Gateway 已共用一个部署脚本，支持交互输入 `1` 或 `2` 选择组件，也保留 `-Component` 自动化入口。本地测试、静态检查、PowerShell 解析和 Linux amd64 交叉编译通过；尚未执行本次生产部署和真实客户端订阅验收。

## 已接受决策

- 模板面向 Clash Verge Rev、Clash Party 等使用 Mihomo 内核并支持远程 YAML 订阅的客户端。
- 完整复用 `override.yaml`，暂不覆盖其中的 `allow-lan`、`external-controller`、TUN、内部 hosts、规则和 Web UI 配置。
- 继续沿用生产服务名 `tfly-sub-gateway`，不改名为 `tfly-gateway`。
- Portal 与 Gateway 使用同一个 `tools/deploy/publish-tfly.ps1`；交互用户用 `1` 或 `2` 选择目标，自动化流程用 `-Component portal|gateway`。
- 生产部署保持显式确认：输入 `DEPLOY` 才继续，直接回车默认取消；受控自动化可用 `-Yes`。

## 实现事实

### 唯一模板源

- 删除了 `gateway/mihomo-template.yaml` 重复副本。
- Gateway 默认模板路径改为 `/etc/tfly-sub-gateway/override.yaml`。
- 安装脚本从发布包同目录或仓库根目录读取 `override.yaml`，安装前备份服务器旧模板。
- Gateway 启动校验覆盖 `proxy-groups`、`rules` 和 `rule-providers`。
- 回归测试将生成文档与解析后的 `override.yaml` 比较，只允许动态 `proxies` 不同。

### 统一部署入口

- 删除旧的 `tools/deploy/publish-tfly-portal.ps1` 当前入口。
- 新增 `tools/deploy/publish-tfly.ps1`，`-Component` 为可选参数。
- 未传 `-Component` 时显示菜单：`1` 部署 Portal，`2` 部署 Gateway 与 `override.yaml`；无效输入会重复询问。
- Portal 分支保留测试、PostgreSQL 集成测试、Linux 构建、哈希校验、备份、重启、公网健康检查和失败回滚。
- Gateway 分支执行测试、静态检查、Linux 构建、二进制与模板双哈希校验、环境文件增量修正、本机健康检查和失败回滚。
- Gateway 回滚同时覆盖二进制、`/etc/tfly-sub-gateway.env` 和 `override.yaml`，不整体覆盖生产环境中的真实上游配置。

## 验证证据

验证环境为 Windows PowerShell、本地项目 `$env:WORKS_ROOT\tfly-portal` 和仓库内捆绑 Go 工具链。

| 验证项 | 结果 |
|---|---|
| Gateway `go test ./...` | 通过 |
| Gateway `go vet ./...` | 通过 |
| Portal `go test ./...` | 通过 |
| Portal `go vet ./...` | 通过 |
| Portal Linux amd64 交叉编译 | 通过 |
| Gateway Linux amd64 交叉编译 | 通过 |
| `publish-tfly.ps1` PowerShell AST 解析 | 通过 |
| 交互 `1/2` 分支与可选 `-Component` 静态检查 | 通过 |
| 临时构建与 Go 缓存清理 | 通过 |

关键文件 SHA-256：

| 文件 | SHA-256 |
|---|---|
| `override.yaml` | `09AE1E7F3C42EBC4E53E25A625B23EDC7FC31E0401E4FF6B35D47044CA1CBDAB` |
| `tools/deploy/publish-tfly.ps1` | `F44C627D47D5C4AC4B9135D9BB067147DEC308C38FE5CDF590CA34117EDF13F5` |
| `gateway/main.go` | `1EAD3B127BA33C9925981646CFDCAE5E0AF94130EA47E6DFC8B3306E23E380B4` |
| `gateway/main_test.go` | `E6C87BC07607074AD3CF150BBA0A54C25F0CC88777CAE69BBB6F04A52BE6C3F3` |

## 状态边界

- **实现完成**：唯一模板源、Gateway 接入、统一部署脚本、交互菜单、文档和本地发布包已经完成。
- **本地验收通过**：源码测试、静态检查、脚本解析和 Linux 构建均已取得通过证据。
- **生产未验收**：本阶段没有连接或修改生产服务器，不能标记为已上线。
- **仍需人工验收**：使用统一脚本选择 `2` 部署 Gateway 后，刷新真实 Portal 订阅，并在 Clash Verge Rev 或 Clash Party 中检查节点、策略组、DNS、TUN 和规则提供器。
- **已接受风险**：`override.yaml` 当前原样保留局域网访问、空控制密钥、全网卡控制接口、TUN 与内部网络配置；观察真实使用情况后再决定是否收紧。

## 相关笔记

- [[TFly Portal项目索引]]
- [[TFly Portal 标准发布与回滚手册]]
- [[2026-08-02 TFly Portal 第一版实施与生产发布验收记录]]
- [[2026-07-31 TFly Portal 项目重命名实施记录]]
