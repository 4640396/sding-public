---
title: 3x-ui 部署、安全与排障
type: guide
status: draft
created: 2026-07-14
updated: 2026-07-15
sensitivity: internal
domain: 运维与排查
tags:
  - 3x-ui
  - Xray
  - Cloudflare
  - TLS
source:
  - "原始笔记：知识库根目录/3x-ui.md（2026-07-14，本次整理后已迁移）"
---
# 3x-ui 部署、安全与排障

> [!info] 关联项目
> 订阅转换网关与自定义订阅页面见 [[3x-ui 订阅系统项目索引]]。

> [!summary] 适用范围
> 这是一份从实际部署与排障中整理出的 3x-ui 运维指南。环境专属的域名、端口、随机路径和凭据已省略；部署时应以当前服务器、云安全组和客户端配置为准。

## 已验证结论

- 管理面板可放在 Cloudflare 橙云之后，源站使用与面板域名匹配的有效证书，SSL/TLS 模式使用 `Full (strict)`。
- Trojan 节点采用 `Trojan + TLS + RAW/TCP` 时可稳定直连；节点域名使用灰云（仅 DNS），客户端开启证书校验。
- 普通 TCP 节点不能依赖 Cloudflare 免费橙云代理。灰云也不会隐藏源站 IP。
- Cloudflare Origin CA 只适合 `Cloudflare → 源站`，不适合浏览器或灰云直连客户端；直连域名应使用受客户端信任的公共证书。
- 面板和节点应分别使用与各自域名匹配的证书。
- 一个入站可以配置多个客户端并以不同凭据区分，不必为每个客户端新建端口。
- 当前环境中的 Reality 方案尚未验证成功，不应替换稳定工作的 Trojan + TLS。

## 推荐部署边界

### 管理面板

- 使用独立域名、随机 Web Base Path 和 Cloudflare 支持的 HTTPS 代理端口。
- Cloudflare SSL/TLS 使用 `Full (strict)`，不要使用 `Flexible`。
- 优先增加 Cloudflare Access 或 Tunnel，减少管理端口直接暴露。
- 证书私钥路径、面板随机路径和登录凭据不得写入普通知识笔记。

Cloudflare 常见 HTTPS 代理端口包括：

```text
443、2053、2083、2087、2096、8443
```

端口在支持列表中不等于当前环境一定可达；仍需检查云安全组、主机防火墙和线路限制。

### Trojan 节点

推荐的稳定组合：

```text
协议：Trojan
传输：RAW/TCP
安全：TLS
DNS：灰云（仅 DNS）
证书校验：开启
```

客户端关键项应保持一致：服务器地址、端口、SNI，以及 `skip-cert-verify: false`。

## 证书签发与使用

使用 Let's Encrypt 进行 HTTP-01 验证时：

- 域名临时切换为灰云通常更容易完成验证。
- TCP 80 必须能从公网访问。
- 自动续期后需要重载或重启 x-ui。
- 面板域名和节点域名分别绑定匹配的证书。

Cloudflare Origin CA 证书不受普通浏览器、Node.js 或直连客户端信任。若直连服务错误地返回 Origin CA，可能出现：

```text
unable to verify the first certificate
Verify return code: 21
```

正确处理方式是为直连域名单独签发公共可信证书，而不是长期跳过证书验证。

## 端口与 TLS 排查顺序

### 1. 检查服务监听

```bash
ss -lntp | grep ':<端口>'
```

### 2. 从客户端测试公网 TCP

```powershell
Test-NetConnection <节点域名> -Port <端口>
```

结果含义：

- `TcpTestSucceeded : True`：DNS、端口和 TCP 路径基本正常。
- `timeout`：优先检查安全组、防火墙、端口配置和线路。
- `connection refused`：目标可达，但对应端口没有服务监听。
- `REALITY authentication failed`：TCP 已连到 Xray，问题位于 Reality 认证层，而非端口开放。

### 3. 检查主机与云侧防火墙

即使主机防火墙策略允许流量，云厂商安全组仍可能拦截端口。排查时需要同时核对两层规则。

### 4. 检查服务端证书

```bash
echo | openssl s_client \
  -connect 127.0.0.1:<端口> \
  -servername <节点域名> \
  -verify_hostname <节点域名> 2>/dev/null | \
  grep -E 'subject=|issuer=|Verify return code'
```

预期结果：

```text
Verify return code: 0 (ok)
```

## Reality 待验证项

目标组合：

```text
VLESS + RAW/TCP + Reality + xtls-rprx-vision
```

若持续出现 `REALITY authentication failed`，依次核对：

1. 客户端公钥是否由当前服务端私钥生成。
2. Short ID 是否完全一致。
3. 客户端 SNI 是否包含在服务端 `serverNames` 中。
4. 客户端是否仍缓存旧配置。
5. 当前客户端内核与服务端 Xray Reality 实现是否兼容。

最小化配置建议：

```text
传输：RAW
目标：选择支持 TLS 1.3 的稳定站点及 443 端口
SNI：与目标站点一致
uTLS：chrome
Xver：0
SpiderX：/
最小/最大客户端版本：留空
Short ID：仅保留一个 16 位十六进制值
Flow：xtls-rprx-vision
```

生成 Short ID：

```bash
openssl rand -hex 8
```

使用 Xray 自带命令生成确定配对的密钥：

```bash
/usr/local/x-ui/bin/xray-linux-amd64 x25519
```

重新生成后，应保存入站、重启 Xray、删除客户端旧配置并重新导入。再使用另一款基于 Xray-core 的客户端交叉测试：

- 只有某个客户端失败：优先判断为客户端内核或配置兼容问题。
- 多个客户端均失败：优先检查服务端密钥、Short ID、SNI 和 Xray 配置。

> [!warning] 当前状态
> Reality 尚未在原环境中验证成功，以上内容是排查路径，不是稳定部署结论。

## 方案选择

| 目标 | 方案 | 当前判断 |
|---|---|---|
| 稳定优先 | Trojan + TLS + RAW/TCP | 已验证，优先使用 |
| 直连抗探测 | VLESS + Reality + Vision | 待验证，不作为主线路 |
| 隐藏源站 | VLESS + WebSocket + TLS + Cloudflare | 可评估延迟、性能和服务条款后使用 |

灰云域名等价于公开源站 IP。更换同一 IP 上的域名或端口不能解决 IP 被封；真正的容错需要独立备用 IP、不同服务商或地区，或兼容 CDN 的线路。

## 安全处置清单

一旦私钥、UUID、密码、订阅 ID、完整订阅链接或认证值出现在截图、聊天或非受控文件中，应视为已经泄露：

- [ ] 撤销并重新签发暴露过的证书。
- [ ] 重新生成 Reality 密钥。
- [ ] 重新生成 UUID、密码、订阅 ID 和认证值。
- [ ] 删除旧客户端配置和订阅缓存。
- [ ] 检查日志与访问记录，确认是否存在异常使用。
- [ ] 后续截图和记录只保留脱敏后的结构信息。

Reality 公钥可以下发给客户端；Reality 私钥、TLS 私钥、UUID、密码、认证值和完整订阅链接不得公开。

## 数据迁移注意事项

3x-ui 常见 SQLite 数据库位于：

```text
/etc/x-ui/x-ui.db
```

若使用 PostgreSQL，可通过 `pg_dump` 导出。数据库备份可能包含 UUID、密码、订阅 ID 和 Reality 私钥，应按高敏感文件管理，不得上传到公共网盘或聊天。

证书文件通常需要单独迁移，或在新服务器上重新签发。恢复后应复核文件权限、证书链、域名匹配和自动续期。

## 后续行动

1. 完成本次已暴露凭据和私钥的轮换，并人工确认。
2. 为管理面板增加 Cloudflare Access 或 Tunnel。
3. 加固 SSH：使用密钥登录、限制来源、关闭密码登录。
4. 仅开放必要端口，复核云安全组与主机防火墙。
5. 使用不同客户端交叉验证 Reality 后，再决定是否投入使用。
6. 准备使用独立 IP 的备用线路。
