---
title: 2026-08-02 TFly Portal 邀请功能生产发布续验记录
type: production-release-follow-up
status: verified
created: 2026-08-02
updated: 2026-08-02
sensitivity: internal
project: TFly Portal
tags:
  - portal
  - invitation
  - deployment
  - 3x-ui
  - ssh
sources:
  - 2026-08-02 本次会话中的发布脚本、systemd、健康检查与生产日志输出
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\xui.go
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\xui_test.go
  - $env:WORKS_ROOT\tfly-portal\tools\deploy\publish-tfly-portal.ps1
  - $env:WORKS_ROOT\tfly-portal\tools\deploy\setup-tfly-portal-ssh-key.ps1
  - "[[2026-08-02 TFly Portal 客户邀请功能需求与验收记录]]"
  - "[[TFly Portal 标准发布与回滚手册]]"
---
# 2026-08-02 TFly Portal 邀请功能生产发布续验记录

> [!summary] 续验结论
> 客户邀请核心功能及 3x-ui 新版 `settings` 响应兼容修复已发布到生产。发布脚本完成测试、构建、哈希核对、原子替换、systemd 检查及本机与公网健康检查，生产二进制 SHA-256 为 `72640857204C33CF749A3E0ED6549B2144CCFB9E9767866D258D68ECFDD54AC9`。后续“候选账号显示上游备注”的构建已通过测试并生成，但尚无生产发布成功回执，不计为已上线。

## 本次生产发布结果

- `go test ./...`：通过。
- `go vet ./...`：通过。
- PostgreSQL 集成测试：通过。
- systemd：`ActiveState=active`、`SubState=running`、`ExecMainStatus=0`、`NRestarts=0`。
- 服务器正式二进制哈希：`72640857204C33CF749A3E0ED6549B2144CCFB9E9767866D258D68ECFDD54AC9`。
- 本机 `/healthz`：通过。
- 公网 `/healthz`：通过。
- 管理员邀请页随后可以读取候选服务账号，证明本次上游兼容修复已生效。

健康检查只证明运行链路正常。真实邀请收信、密码设置、流量与有效期一致性仍应按发布手册逐项做浏览器人工验收。

## 生产故障与修复

### 3x-ui `settings` 类型变化

生产日志曾出现：

```text
service account listing failed: json: cannot unmarshal object into Go struct field .obj.settings of type string
```

根因是上游新版本可能直接返回 JSON 对象，而旧兼容代码只接受 JSON 编码字符串。Portal 已改为先接收 `json.RawMessage`，再兼容解析字符串和对象两种形式，并新增对象形式回归测试。该修复随上述生产哈希发布。

### Windows 到远端 Bash 的 CRLF

首次自动发布虽然已替换并启动服务，但最后的远端哈希命令把路径识别为带 `\r` 的文件名。根因是 Windows PowerShell 将远端脚本以 CRLF 传给 Bash。发布脚本已改为先去除 `\r`，再以 UTF-8 Base64 传输并在远端解码执行。

服务重启窗口内第一次 `curl` 短暂连接失败属于重试过程；只有最终重试仍失败，或 systemd 状态异常时，才判定发布失败。

## SSH 免密发布

- 已使用独立 Ed25519 部署密钥完成一次性配置。
- 验证输出为 `key-auth-ok`，后续发布脚本会自动使用该密钥并禁止回退到其他身份。
- 首版安装逻辑因远端 shell 对公钥文本分词导致失败，已改为 Base64 传输公钥后追加到 `authorized_keys`。
- 服务器密码只在本机 SSH 客户端中输入一次，脚本不保存密码；私钥仅保存在本机用户的 `.ssh` 目录，不进入代码仓库或 Vault。

## 后续界面增强

邀请页已完成以下本地增强：

- 原生下拉框改为与后台一致的深色样式，并增加静态资源版本参数，减少浏览器或 CDN 旧缓存影响。
- 候选服务账号读取上游 `comment` 字段，优先显示“备注 · 服务账号”，没有备注时只显示服务账号。
- 字符串形式和对象形式 `settings` 的回归测试均覆盖备注字段。

该增强构建 SHA-256 为 `79A94D0E593E438076EEF39176783E5985DA7A47E34684797616B7AB1DB46DCC`。当前只有构建与测试证据，没有对应的完整生产发布成功输出，因此状态为“待发布确认”。

## 安全边界

- 本记录不保存服务器地址、密码、API Key、私钥、真实客户邮箱或订阅凭据。
- 自动发布继续保留显式 `DEPLOY` 确认；只有在其他受控流程已经确认时才使用 `-Yes`。
- SSH 免密只减少重复输入，不改变生产替换、健康检查和回滚要求。

## 关联记录

- [[TFly Portal项目索引]]
- [[2026-08-02 TFly Portal 客户邀请功能需求与验收记录]]
- [[TFly Portal 标准发布与回滚手册]]

