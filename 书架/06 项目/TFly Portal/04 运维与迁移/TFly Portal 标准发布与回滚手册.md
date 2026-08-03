---
title: TFly Portal 标准发布与回滚手册
type: operations-runbook
status: verified
created: 2026-08-02
updated: 2026-08-02
sensitivity: internal
project: TFly Portal
tags:
  - portal
  - deployment
  - systemd
  - rollback
  - nginx
sources:
  - $env:WORKS_ROOT\tfly-portal\releases\tfly-portal.service
  - $env:WORKS_ROOT\tfly-portal\releases\tfly-portal-production.env.example
  - $env:WORKS_ROOT\tfly-portal\portal\internal\portal\migrations
  - $env:WORKS_ROOT\tfly-portal\tools\deploy\publish-tfly.ps1
  - $env:WORKS_ROOT\tfly-portal\override.yaml
  - $env:WORKS_ROOT\tfly-portal\gateway\main.go
  - $env:WORKS_ROOT\tfly-portal\gateway\main_test.go
  - $env:WORKS_ROOT\tfly-portal\tools\deploy\setup-tfly-portal-ssh-key.ps1
  - "[[2026-08-02 TFly Portal 第一版实施与生产发布验收记录]]"
  - "[[2026-08-02 TFly Portal 订阅模板与统一部署实施验收记录]]"
---
# TFly Portal 标准发布与回滚手册

> [!summary] 适用范围
> 本手册用于将本地验收通过的 TFly Portal 或 tfly-sub-gateway 发布到现有生产服务器。Portal 前端模板和静态资源已嵌入 Go 二进制；Gateway 发布同时携带根目录唯一模板 `override.yaml`。统一脚本通过交互菜单或 `-Component` 参数选择目标，不会在一次发布中同时重启两个服务。

## 固定部署结构

| 用途 | 路径或服务 |
|---|---|
| 本地项目 | `$env:WORKS_ROOT\tfly-portal` |
| Portal 本地发布文件 | `$env:WORKS_ROOT\tfly-portal\releases\tfly-portal-linux-amd64` |
| Gateway 本地发布文件 | `$env:WORKS_ROOT\tfly-portal\releases\tfly-sub-gateway-linux-amd64` |
| Portal 上传暂存区 | `/root/tfly-deploy/portal` |
| Gateway 上传暂存区 | `/root/tfly-deploy/gateway` |
| Portal 正式程序 | `/opt/tfly-portal/tfly-portal` |
| Gateway 正式程序 | `/opt/tfly-sub-gateway/tfly-sub-gateway` |
| Portal 环境文件 | `/etc/tfly-portal.env` |
| Gateway 环境文件 | `/etc/tfly-sub-gateway.env` |
| Gateway 模板 | `/etc/tfly-sub-gateway/override.yaml` |
| systemd 服务 | `tfly-portal.service`、`tfly-sub-gateway.service` |
| 本机监听 | Portal `127.0.0.1:18081`；Gateway `127.0.0.1:18080` |
| 公网入口 | `https://sub.tfly.org` |

`/root/tfly-deploy/portal` 与 `/root/tfly-deploy/gateway` 只是可重复使用的上传暂存区。先上传到暂存区可以完成哈希核对和权限检查，不会覆盖正在运行的正式程序；正式运行目录仍位于对应的 `/opt` 目录。

## 一、发布前检查

### 首次配置独立 SSH 发布密钥

每台发布电脑只需要执行一次：

```powershell
& "$env:WORKS_ROOT\tfly-portal\tools\deploy\setup-tfly-portal-ssh-key.ps1" -Server '<SERVER>'
```

首次安装时服务器密码由本机 SSH 客户端请求一次，脚本不读取也不保存密码。出现 `key-auth-ok` 和“SSH key authentication is ready”后即配置成功。发布脚本会自动发现本机用户 `.ssh` 目录中的专用部署密钥；也可通过 `-IdentityFile` 显式指定。

> [!warning] 私钥边界
> 不要把私钥内容发送到聊天、复制到 Vault、提交到代码仓库或上传到服务器。设置密钥口令更安全，但每次新登录会话需要先用 `ssh-add` 将密钥加入 `ssh-agent`；留空则可以完全免交互发布。

### 推荐：一条命令自动发布

在 Windows PowerShell 中直接复制执行以下命令。`Process` 作用域只对当前 PowerShell 窗口生效，关闭窗口后自动失效，不修改系统或当前用户的永久执行策略：

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
Set-Location "$env:WORKS_ROOT\tfly-portal"
.\tools\deploy\publish-tfly.ps1 -Server '<SERVER>'
```

脚本显示组件菜单后，输入 `1` 部署 Portal，输入 `2` 部署 Gateway 与 `override.yaml`。选定组件后仍需输入 `DEPLOY` 二次确认；直接回车或其他输入默认取消。自动化流程可显式传入 `-Component portal|gateway -Yes`，不把生产部署设为默认动作。

看到确认提示后输入：

```text
DEPLOY
```

如果出现“在此系统上禁止运行脚本”或 `PSSecurityException`，说明 PowerShell 在脚本启动前被执行策略拦截；先执行上面的 `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass`，再重新运行发布命令。不要为此设置全局 `Unrestricted`。

也可以不改变当前会话状态，使用单条命令临时启动：

```powershell
powershell.exe -ExecutionPolicy Bypass -File .\tools\deploy\publish-tfly.ps1 -Server '<SERVER>'
```

以下参数化命令用于服务器地址发生变化时：

```powershell
& "$env:WORKS_ROOT\tfly-portal\tools\deploy\publish-tfly.ps1" -Server '<SERVER>'
```

脚本要求选择组件并输入 `DEPLOY` 后继续，自动完成对应模块的测试、构建、哈希核对、上传、备份替换、重启和健康检查。Portal 失败时恢复 `tfly-portal.previous`；Gateway 失败时恢复旧二进制、环境文件和 `override.yaml`。已在其他受控流程中确认时可显式增加 `-Yes`；紧急跳过测试可增加 `-SkipTests`，但必须记录原因。

Gateway 自动部署会把根目录 `override.yaml` 安装到 `/etc/tfly-sub-gateway/override.yaml`，并增量修正现有 `/etc/tfly-sub-gateway.env` 中的 `TEMPLATE_FILE`。不得用示例环境文件整体覆盖生产环境，因为其中包含真实上游配置。

自动发布将远端 Bash 脚本去除 Windows CR 字符后，以 UTF-8 Base64 传输并解码执行，避免远端路径末尾混入 `\r`。服务重启时首次健康检查可能短暂连接失败，脚本会重试；以最终 systemd、哈希、本机和公网检查结果为准。

以下内容是 Portal 分支的人工执行版本，用于排障和回滚；Gateway 日常发布优先使用统一脚本，失败边界见本手册前述说明和 [[2026-08-02 TFly Portal 订阅模板与统一部署实施验收记录]]。

在本机 PowerShell 中执行：

```powershell
$portalRoot = Join-Path $env:WORKS_ROOT 'tfly-portal'
Set-Location $portalRoot

Push-Location portal
go test ./...
go vet ./...
go test -tags=integration ./internal/portal -run TestPortalAuthenticationSmokeWithPostgres -count=1
Pop-Location
```

只有三项都通过才继续构建。集成测试会启动临时 PostgreSQL，覆盖认证、邀请、重发、撤销、服务绑定和订阅 Token 等关键路径。

## 二、构建 Linux 发布文件

在项目根目录的 PowerShell 中执行：

```powershell
$env:GOOS = 'linux'
$env:GOARCH = 'amd64'
$env:CGO_ENABLED = '0'
$env:GOFLAGS = '-buildvcs=false'

go build -trimpath -ldflags='-s -w' `
  -o .\releases\tfly-portal-linux-amd64 `
  .\portal\cmd\portal

Get-FileHash .\releases\tfly-portal-linux-amd64 -Algorithm SHA256
```

记录输出的 SHA-256。不要根据文件名判断版本，服务器端核对必须使用同一个哈希。

清理本次命令设置的构建变量：

```powershell
Remove-Item Env:GOOS, Env:GOARCH, Env:CGO_ENABLED, Env:GOFLAGS -ErrorAction SilentlyContinue
```

## 三、上传到服务器暂存区

将 `<SERVER>` 替换为当前可用的 SSH 主机名或 Tailscale 地址：

```powershell
ssh root@<SERVER> "mkdir -p /root/tfly-deploy/portal"

scp .\releases\tfly-portal-linux-amd64 `
  root@<SERVER>:/root/tfly-deploy/portal/tfly-portal-linux-amd64.new
```

随后登录服务器：

```powershell
ssh root@<SERVER>
```

在服务器中检查上传文件：

```bash
ls -lh /root/tfly-deploy/portal/tfly-portal-linux-amd64.new
sha256sum /root/tfly-deploy/portal/tfly-portal-linux-amd64.new
```

服务器哈希必须与本机记录完全一致。不同则停止发布并重新上传。

## 四、检查配置与依赖

不要用示例环境文件覆盖生产文件。只检查当前版本新增的变量，并保留生产凭据：

```bash
grep '^PORTAL_INVITATION_TTL=' /etc/tfly-portal.env || \
  printf '\nPORTAL_INVITATION_TTL=24h\n' >> /etc/tfly-portal.env

systemctl is-active postgresql
systemctl is-active tfly-sub-gateway
curl -fsS http://127.0.0.1:18080/healthz
```

> [!warning] 敏感配置
> `/etc/tfly-portal.env` 包含数据库、SMTP 和上游服务凭据。不要把它复制到聊天、知识库或代码仓库，也不要用 `tfly-portal-production.env.example` 整体替换它。

## 五、备份并替换二进制

先把新文件安装到正式目录中的临时文件，再停止服务和原子替换：

```bash
install -o tfly-sub -g tfly-sub -m 0755 \
  /root/tfly-deploy/portal/tfly-portal-linux-amd64.new \
  /opt/tfly-portal/tfly-portal.new

systemctl stop tfly-portal
cp -a /opt/tfly-portal/tfly-portal /opt/tfly-portal/tfly-portal.previous
mv -f /opt/tfly-portal/tfly-portal.new /opt/tfly-portal/tfly-portal
systemctl start tfly-portal
```

不要直接通过 SCP 覆盖正在运行的 `/opt/tfly-portal/tfly-portal`。`.new` 文件用于先完成上传和核验，`mv` 只在停服后的最后一步替换正式文件。

Portal 启动时自动执行数据库迁移。本次邀请功能迁移只扩展用户状态和操作 Token 类型；发布前的集成测试已经验证空库执行与重复执行。

## 六、发布后验收

先检查进程和本机链路：

```bash
systemctl status tfly-portal --no-pager -l
systemctl show tfly-portal \
  -p ActiveState -p SubState -p Result -p ExecMainStatus -p NRestarts
ss -lntp | grep ':18081'
curl -i http://127.0.0.1:18081/healthz
journalctl -u tfly-portal -n 50 --no-pager
```

预期结果：

- `ActiveState=active`、`SubState=running`、`ExecMainStatus=0`；
- `127.0.0.1:18081` 有 `tfly-portal` 监听；
- 本机健康检查返回 `HTTP 200` 和 `ok`；
- 日志没有迁移、数据库、SMTP 或上游登录错误。

再检查公网链路：

```bash
curl -i https://sub.tfly.org/healthz
```

公网健康检查通过后，再用浏览器做本次版本的人工验收：

1. 管理员登录 `/admin`，进入“邀请客户”。
2. 确认列表只显示启用且未绑定的服务账号。
3. 向测试邮箱发送邀请并确认实际收信。
4. 使用邮件链接设置密码，登录用户中心。
5. 确认服务状态、流量、有效期和订阅信息正确。
6. 另做一次“重发邀请”，确认旧链接失效；必要时验证“撤销邀请”。

## 七、失败回滚

如果新服务无法启动或健康检查失败：

```bash
systemctl stop tfly-portal
cp -a /opt/tfly-portal/tfly-portal.previous /opt/tfly-portal/tfly-portal
chown tfly-sub:tfly-sub /opt/tfly-portal/tfly-portal
chmod 0755 /opt/tfly-portal/tfly-portal
systemctl start tfly-portal

systemctl status tfly-portal --no-pager -l
curl -i http://127.0.0.1:18081/healthz
journalctl -u tfly-portal -n 80 --no-pager
```

> [!important] 数据库回滚边界
> 二进制回滚不等于数据库回滚。每次包含数据库迁移的发布，都必须在发布前判断旧二进制是否兼容迁移后的结构。本次迁移是扩展约束，旧版本可以继续启动，但无法管理新增的待邀请账号；因此回滚后应暂停邀请操作并尽快修复或重新发布。

## 八、什么时候需要发布其他文件

日常功能发布通常只上传 Portal 二进制。

- `tfly-portal.service`：只有 systemd 运行参数或安全限制变化时才更新，更新后执行 `systemctl daemon-reload`。
- `/etc/tfly-portal.env`：只有配置项变化时手工增量编辑，禁止整体覆盖。
- Nginx 配置：只有域名、路由或反向代理变化时才更新；修改后必须先执行 `nginx -t`，成功后才能 `systemctl reload nginx`。
- Gateway 二进制：只有订阅转换逻辑变化时单独发布，不随 Portal 日常发布自动替换。

Nginx 备份不得放在 `sites-enabled`，否则仍可能被加载并产生重复监听。应放到未被 Nginx `include` 的备份目录。

## 九、发布记录最小模板

每次生产发布至少记录以下信息：

```text
发布时间：
发布内容：
本地测试：通过 / 未通过
集成测试：通过 / 未执行及原因
二进制 SHA-256：
服务器启动状态：
本机 /healthz：
公网 /healthz：
人工业务验收：
回滚文件：/opt/tfly-portal/tfly-portal.previous
未验证项：
```

## 关联记录

- [[TFly Portal项目索引]]
- [[2026-08-02 TFly Portal 第一版实施与生产发布验收记录]]
- [[2026-07-31 3x-ui 私有管理面收口与订阅故障复盘]]
- [[2026-08-02 TFly Portal 邀请功能生产发布续验记录]]
