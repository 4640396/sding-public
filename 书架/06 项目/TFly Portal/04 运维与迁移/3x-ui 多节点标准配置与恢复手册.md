---
title: 3x-ui 多节点标准配置与恢复手册
type: operations-runbook
status: accepted
created: 2026-07-31
updated: 2026-07-31
sensitivity: internal
project: TFly Portal
tags:
  - 3x-ui
  - multi-node
  - Tailscale
  - TLS
  - UFW
  - recovery
sources:
  - "[[2026-07-31 3x-ui 备用节点与官方多节点接入验收记录]]"
  - "[[2026-07-31 3x-ui 私有管理面收口与订阅故障复盘]]"
  - "[[2026-07-31 3x-ui 备用节点数据库恢复续验记录]]"
  - https://github.com/mhsanaei/3x-ui/blob/main/docs/architecture.md
  - https://github.com/MHSanaei/3x-ui/blob/main/install.sh
  - https://tailscale.com/kb/1153/enabling-https
---
# 3x-ui 多节点标准配置与恢复手册

> [!summary]
> 本手册把主节点、备用节点和以后新增节点统一为“管理面只走 Tailscale，数据面只开放明确代理端口，主节点使用 HTTPS、标准 CA 校验和每节点独立 API Token 控制子节点”的标准。本文是已接受的运维标准，不代表每个现有节点都已按本手册完成验收。

## 适用范围

- 主节点：唯一的集中管理入口，维护用户、入站、订阅和远程节点关系。
- 备用节点：当前已接入的远程 3x-ui 节点，保留独立 Xray 数据面。
- 新节点：以后增加的第三个及更多远程节点，必须复用同一控制面基线。
- 不在 Vault 保存真实 IP、MagicDNS 全名、管理路径、用户名、密码、API Token、UUID、订阅 ID、REALITY 私钥或 Short ID。

## 角色与不变量

| 项目 | 主节点 | 远程节点 |
|---|---|---|
| 3x-ui | 集中管理 | 独立 3x-ui 实例 |
| 管理网络 | Tailscale | Tailscale |
| 面板协议 | HTTPS | HTTPS |
| TLS | 受信任证书 | 节点独立 MagicDNS 证书 |
| 认证 | 管理员登录 | 每节点独立、可撤销 API Token |
| 数据面 | 按需 | 仅开放实际代理入站端口 |
| 数据库 | 生产后端按现状管理 | 默认独立 SQLite，扩容前再决策 PostgreSQL |
| 订阅 | 统一公开入口 | 不直接暴露管理域名或面板路径 |

必须长期保持：

1. 管理地址与公网代理地址分离；MagicDNS 只用于节点控制，公网节点域名只用于客户端数据面。
2. 一个远程节点使用一个专用 Token；不得在多个节点之间复用管理员密码或 Token。
3. 面板、SSH 和节点 API 不对公网放行；只允许 Tailscale 接口访问。
4. 每个代理入站只开放自己的 TCP 或 UDP 端口，不按大范围端口段放行。
5. 端口、证书、监听地址、Web Base Path、API Token 和数据库必须作为同一组恢复依赖检查，不能只恢复数据库。
6. 任何密钥或 Token 一旦出现在截图、聊天或普通日志中，立即轮换。

## 主节点标准基线

### 管理面

- 仅通过 Tailscale 访问面板和 SSH。
- 主节点启用 Tailscale DNS，并能解析所有远程节点 MagicDNS 名称。
- 节点连接使用 `https`、标准 CA 校验 `verify` 和允许私有地址。
- 每个远程节点保留独立节点条目、独立 Token 和清晰备注。
- 节点名称使用稳定逻辑名，如 `tfly-node2`、`tfly-node3`；不要把临时 IP 写入名称。

### 订阅与入站

- 新入站优先在主节点创建并选择部署目标。
- 分享地址使用独立公网节点域名，不使用 Tailscale 管理域名。
- 订阅排序应稳定、唯一，便于客户端刷新后识别新增节点。
- 修改端口后同时核对主面板入站、远程 Xray 监听、UFW、云安全组和订阅输出。

## 新增远程节点标准流程

### 一、建立回滚点

1. 确认操作系统、3x-ui 和 Xray 版本。
2. 备份远程节点数据库、证书、Xray 配置和系统服务配置。
3. 记录备份文件哈希；备份放受控目录，不上传公共仓库或普通网盘。
4. 保留当前 SSH 会话，后续启用防火墙前再建立第二个独立会话。

### 二、加入 Tailnet

1. 从官方发行版仓库安装 Tailscale。
2. 验证 `tailscaled` 为 active、节点已加入正确 Tailnet。
3. 从管理员终端验证 MagicDNS、`tailscale ping` 和私网 SSH。
4. 在修改公网 SSH 或 UFW 前，保持第二个 Tailscale SSH 会话在线。

### 三、固定私有面板

1. 为该节点选择统一管理端口；当前标准端口为 `2083`。
2. 使用随机 Web Base Path；不要复用已经暴露的路径。
3. 将 3x-ui 面板仅绑定该节点的 Tailscale IPv4。
4. 关闭不需要的远程节点原生订阅服务。
5. 使用底层二进制修改设置，避免管理脚本不转发参数：

```bash
/usr/local/x-ui/x-ui setting -port 2083
TS_IP="$(tailscale ip -4 | head -n 1)"
/usr/local/x-ui/x-ui setting -listenIP "$TS_IP"
```

### 四、配置 MagicDNS HTTPS

1. 在 Tailnet 启用 HTTPS Certificates。
2. 为该节点完整 MagicDNS 名称签发证书。
3. 证书和私钥放在受控路径，权限分别建议为 `644` 和 `600`。
4. 验证 SAN、签发者和有效期。
5. 用 3x-ui 底层二进制配置证书：

```bash
/usr/local/x-ui/x-ui cert \
  -webCert /path/to/fullchain.pem \
  -webCertKey /path/to/privkey.pem
x-ui restart
```

6. 从远程节点本机、主节点和管理员终端分别完成不使用 `-k` 的 HTTPS 校验。
7. 根路径返回 `404` 可以证明 TLS 与 Web 服务可达；随机管理路径应显示登录页。

> [!warning]
> 首次签发不等于长期生产就绪。必须补充自动续期、续期后重载 3x-ui、到期告警和失败回滚。

### 五、收口防火墙

1. UFW 默认拒绝入站、允许出站、禁止 routed 流量。
2. 放行 Tailscale 接口和 Tailscale 直连 UDP。
3. 不为公网单独放行 SSH、面板端口和管理 API。
4. 仅按实际协议放行代理数据面端口，并同步云安全组。
5. 端口迁移时先开放新端口并验收，再确认旧端口无监听、无用户、无网站依赖后关闭旧规则。

### 六、创建节点专用 Token

1. 在远程面板的认证设置中创建独立 Token，名称使用可识别标签，如 `central-panel-a` 或 `tfly-main-panel`。
2. Token 明文通常只显示一次，立即存入受控凭据管理位置。
3. 不在截图、聊天、Vault 或普通日志中保存 Token。
4. 数据库恢复后必须默认假设 Token 已回退；重新创建并在主节点更新。

### 七、在主节点注册

主节点节点表单统一使用：

- 协议：`https`。
- 地址：远程节点完整 MagicDNS 名称。
- 端口：`2083`。
- 基础路径：远程节点当前随机 Web Base Path。
- 允许私有地址：启用。
- TLS 校验：`verify`，使用系统 CA。
- API Token：该节点专用 Token。
- 连接出站：默认直接连接，除非多跳方案已单独验收。
- 入站导入：新节点默认“选定的入站”；已有历史用户的节点按迁移策略决定是否导入全部。

点击“测试连接”后，至少确认面板版本、Xray 版本、运行时长、延迟和在线心跳，再保存。

## 既有入站与用户迁移

### 新节点没有历史用户

- 不导入历史入站。
- 在主节点创建一个测试入站和有限额测试客户端。
- 验收后再逐步增加正式入站。

### 节点已有未到期用户

- 不删除原入站和客户端。
- 可使用“全部入站”导入，但导入前必须备份数据库并检查端口冲突。
- 主节点已经管理的同端口入站不得与导入记录同时启用。
- 验证 UUID 或密码、流量额度、到期时间、TLS/REALITY 参数和订阅链接是否保留。
- 旧入站进入过渡期：不再新增用户，等待到期或通知用户刷新订阅后再下线。
- 防火墙按仍有有效用户的旧端口逐项保留，不开放无用户端口。

## 数据库恢复标准流程

> [!important]
> 恢复数据库会同时恢复面板端口、监听地址、Web Base Path、证书路径、API Token 和入站状态。数据库恢复完成不代表节点恢复完成。

1. 恢复前备份当前数据库和证书，并记录哈希。
2. 恢复数据库后先保持 SSH，不立即修改公网防火墙。
3. 读取实际配置：

```bash
x-ui settings
/usr/local/x-ui/x-ui setting -show true
/usr/local/x-ui/x-ui setting -getCert true
```

4. 恢复标准端口、Tailscale 监听、Web Base Path 和 MagicDNS 证书引用。
5. 重启并检查：

```bash
x-ui restart
ss -lntp
journalctl -u x-ui -n 100 --no-pager
```

6. 验证证书 SAN、有效期和证书/私钥匹配。
7. 在远程面板创建新的节点专用 API Token，并更新主节点节点条目。
8. 主节点出现 `HTTP 404 from remote panel` 时，若 TLS 和端口可达，优先检查 Token；3x-ui 会对未认证的普通 API 请求返回 `404`。
9. 核对恢复后的入站启用状态和端口监听，避免旧数据库重新启用历史入站。
10. 完成主节点心跳、远程操作和客户端实测后，才标记恢复通过。

## 节点验收清单

### 管理链路

- [ ] Tailscale 与 MagicDNS 在线。
- [ ] 私网 SSH 可用，公网 SSH 不可达。
- [ ] 面板仅监听 Tailscale 地址和标准端口。
- [ ] HTTPS 可通过系统 CA 严格校验。
- [ ] 主节点测试连接成功并显示版本、运行时长和延迟。
- [ ] 每节点 Token 独立且已安全保存。

### 数据面

- [ ] Xray 只监听已登记入站端口。
- [ ] UFW 与云安全组规则一致。
- [ ] 公网节点域名解析到正确服务器，不经不兼容的代理层。
- [ ] 订阅输出包含正确地址、端口、协议和安全参数。
- [ ] 一个有限额测试客户端完成真实访问。

### 恢复与运维

- [ ] 数据库、证书和 Xray 配置存在受控备份与哈希。
- [ ] 数据库恢复后完成设置、证书、Token 和入站状态复核。
- [ ] 证书自动续期、重载和告警已验收。
- [ ] 节点失联、恢复、禁用和删除已演练。
- [ ] 单节点停止时公开订阅与客户端行为符合预期。

## 当前未完成边界

- 备用节点数据库恢复后的新 API Token 尚未在主节点完成重连验收。
- Tailscale HTTPS 证书自动续期和 3x-ui 重载尚未建立。
- 新增第三节点尚未按本手册完整演练。
- 多节点故障切换、公开订阅聚合和节点生命周期操作仍需验收。
- 历史入站“全部导入”后的客户端、流量和到期字段保真仍需逐项验证。

## 关联记录

- [[TFly Portal项目索引]]
- [[2026-07-31 3x-ui 备用节点与官方多节点接入验收记录]]
- [[2026-07-31 3x-ui 备用节点数据库恢复续验记录]]
- [[2026-07-31 3x-ui 私有管理面收口与订阅故障复盘]]
- [[2026-07-31 3x-ui 多节点与管理面架构方案]]
