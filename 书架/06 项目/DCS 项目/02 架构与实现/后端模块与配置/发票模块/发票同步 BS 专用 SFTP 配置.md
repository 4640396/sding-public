---
title: "发票同步 BS 专用 SFTP 配置"
type: project-knowledge
status: verified
created: 2026-07-15
updated: 2026-07-16
sensitivity: internal
project: DCS
repository: dcs-global-vehicle
---
# 发票同步 BS 专用 SFTP 配置

## 背景与结论

全球整车后端原先由发票同步与 Santander 信贷任务共同注入默认 `FileSftpServer`，都使用公共 `sftp.config`。如果为了 BS 发票同步直接修改公共 SFTP，可能同时改变信贷文件上传等其他功能的连接目标。

改造结论：为发票同步建立独立的 `InvoiceBsFileSftpServer`，仅发票同步流程使用；Santander 等其他功能继续使用默认 `FileSftpServer`。

## 配置结构

发票文件格式与专用 SFTP 统一归入现有 `syn.bs.invoice`：

```yaml
syn:
  bs:
    invoice:
      file:
        type: txt # 按 BS 接口要求配置 txt 或 xls
      sftp:
        config:
          host: <host>
          username: <username>
          password: <从安全配置源注入>
          port: <port>
          time-out: 10000
          privatekey-connection-mode: false
```

Java 配置绑定前缀：

```java
@ConfigurationProperties(prefix = "syn.bs.invoice.sftp.config")
```

真实连接地址、账号和凭据应放在 Apollo 或其他安全配置源中，不写入知识库，也不建议提交到 Git。

## 代码改造范围

新增专用 Bean：

```text
dcs-service/src/main/java/com/smil/globalvehicle/invoice/applicationservice/config/InvoiceBsFileSftpServer.java
```

以下发票同步操作改为注入并使用 `InvoiceBsFileSftpServer`：

- 发票文件上传。
- 取消发票文件上传。
- 取消前检查 Pending 文件是否存在。
- 删除 Pending 文件。

主要调用位置：

```text
dcs-service/src/main/java/com/smil/globalvehicle/invoice/applicationservice/SynInvoiceExecuteService.java
dcs-service/src/main/java/com/smil/globalvehicle/invoice/applicationservice/SynInvoiceApplicationService.java
```

以下功能保持使用公共 SFTP，不应切换到发票专用 Bean：

```text
dcs-service/src/main/java/com/smil/globalvehicle/credit/applicationservice/job/SantanderHistoryReversedInvoiceDetectionJob.java
```

## 配置说明

- `file.type=txt`：生成 TXT 发票同步文件。
- `file.type=xls`：生成 Excel 发票同步文件，必须以 BS 接口实际要求为准。
- `time-out=10000`：保持原连接超时效果，单位由现有 `FileSftpServer` 实现定义。
- `privatekey-connection-mode=false`：使用用户名和密码认证；私钥认证时需按组件要求改为 `true` 并配置私钥。
- 新旧 SFTP 可以先配置成相同连接参数，以保持行为一致；后续只修改 `syn.bs.invoice.sftp.config` 即可独立切换发票 BS SFTP。

## 验证与发布检查

- 静态检查应确认发票同步代码不再注入普通 `FileSftpServer`。
- 静态检查应确认 Santander 仍注入普通 `FileSftpServer`。
- 检查各部署环境或 Apollo 均存在 `syn.bs.invoice.sftp.config` 完整配置。
- 联调时分别验证发票上传、取消上传、Pending 文件检查和删除。
- Maven 编译曾因本地缺少 `dcs-excel-bean` 与 SAP JCo 私有依赖而未进入本次代码编译阶段，完整构建需要私服或本地依赖恢复后补做。

## `host must not be null` 排查记录

### 现象

发票同步上传时报错：

```text
com.jcraft.jsch.JSchException: host must not be null
```

`SynInvoiceExecuteService` 会先记录“上传发票文件失败”，外层调用随后记录“发送发票同步信息失败”。两条日志通常来自同一次上传异常，不代表发生了两个独立故障。

### 根因判断

该异常发生在建立网络连接之前，含义是 `InvoiceBsFileSftpServer` 已被 Spring 注入并调用，但其 `host` 属性没有绑定到值。它不是端口不通、账号错误或远端目录不存在导致的异常。

当前 Java Bean 读取的完整前缀是：

```text
syn.bs.invoice.sftp.config
```

需要重点排除以下情况：

1. Apollo 仍使用改造前的 `sync.invoice.bs...` 前缀，和代码的 `syn.bs.invoice...` 不一致。
2. 配置只写在 `bootstrap-sit.yml`，实际实例使用的是 `pre` 或 `prd` profile。
3. Apollo 对应环境或集群没有发布发票专用 SFTP 配置。
4. 配置已修改但实例没有刷新到最新值，需要确认 Apollo 发布状态并视情况重启实例。

仓库核查时，SIT profile 已包含 `syn.bs.invoice.sftp.config`，而 pre、prd profile 本身没有该配置，仅包含 Apollo 入口。因此 pre/prd 必须由对应环境的 Apollo 提供完整属性，否则 `host` 会保持为 `null`。

### Apollo 检查清单

当前部署环境应存在并发布以下属性；值必须从安全配置源获取，不在文档中记录：

```properties
syn.bs.invoice.sftp.config.host=<host>
syn.bs.invoice.sftp.config.username=<username>
syn.bs.invoice.sftp.config.password=<secret>
syn.bs.invoice.sftp.config.port=<port>
syn.bs.invoice.sftp.config.time-out=10000
syn.bs.invoice.sftp.config.privatekey-connection-mode=false
```

排查顺序：

1. 确认报错实例的 active profile、Apollo 环境、集群和 namespace。
2. 确认属性前缀严格为 `syn.bs.invoice.sftp.config`。
3. 确认配置已发布，而不只是保存在草稿中。
4. 重启或刷新实例后再次触发同步。
5. `host must not be null` 消失后，再根据后续异常判断网络、认证或远端目录问题。

### 配置建议

Apollo 中建议直接配置发票专用 SFTP 的实际参数，不再通过 `${sftp.config.host}` 等占位符间接引用公共 SFTP。这样既能避免配置加载顺序或 profile 差异造成空值，也能真正隔离发票同步与 Santander 等其他功能。

## BS 回执与发票状态变更时点

### SFTP 目录职责

旧 Mexico 实现中，请求文件和回执文件使用不同目录：

```text
/DCS/SMMX/INVOICE/Pending/
    DCS 上传、等待 BS 处理的开票请求。

/DCS/SMMX/INVOICE/Processed/
    BS 已取走或处理过的开票请求。

/DCS/SMMX/Response_Invoice/Pending/
    BS 返回、等待 DCS 消费的开票回执。

/DCS/SMMX/Response_Invoice/Processed/
    DCS 已成功消费并归档的开票回执。
```

请求文件进入 `/DCS/SMMX/INVOICE/Processed/` 只代表 BS 处理了请求文件，不是 DCS 更新开票业务状态的触发点。

### 旧 Mexico 状态更新流程

旧 Mexico 后端通过 XXL-Job Handler `SynInvoiceRespJob` 消费回执，执行频率由 XXL-Job 调度中心配置，代码仓库不包含具体调度周期。

处理过程：

1. 扫描 `/DCS/SMMX/Response_Invoice/Pending/`。
2. 下载并解析 `Response_...` 回执文件。
3. 读取 DCS 发票号、本地发票号和发票链接。
4. 校验当前 `syncStatus` 为 `Send Invoice Success`。
5. 更新本地发票号、发票链接和 `syncStatus=Get Invoice Success`。
6. 将 BS 发票业务状态更新为 `INVOICED=3`。
7. 数据库事务成功后，将回执移动到 `/DCS/SMMX/Response_Invoice/Processed/`。

旧 Mexico 将 `INVOICED=3` 写入 `invoice_status`。状态拆分后的 global 设计应写入：

```java
invoiceDao.setBsInvoiceStatus(BsInvoiceStatus.INVOICED.getCode());
```

此时 `invoiceStatus` 仍保持 `CREATED=1`，分别表达 DCS 发票本体状态和 BS 开票业务状态。

### Global 回执任务迁移结果

2026-07-15 已将旧 Mexico 的 `SynInvoiceRespJob` 迁移到 Global，代码位置：

```text
dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/invoice/applicationservice/job/SynInvoiceRespJob.java
```

同时在 `InvoiceConstant` 中补充：

```text
SYN_INVOICE_RES_FILE_PATH_PENDING
SYN_INVOICE_RES_FILE_PATH_PROCESSED
```

迁移保持 Mexico 原有目录扫描、文件级业务锁、Excel 解析、开票/取消分支、事务处理和成功后归档流程。Global 必要适配为：

- 使用发票专用 `InvoiceBsFileSftpServer`，不使用公共 `FileSftpServer`。
- 普通开票回执写入 `bsInvoiceStatus=BsInvoiceStatus.INVOICED`，`invoiceStatus` 保持 `CREATED`。
- 取消请求发出后由既有逻辑写入 `bsInvoiceStatus=CANCELLING`。
- 取消回执仅更新当前 `bsInvoiceStatus=CANCELLING` 的记录，然后调用既有取消领域逻辑。
- 操作人改为 `global_veh_admin`。

### 部署与联调检查

- 在 XXL-Job 调度中心创建或启用 Handler `SynInvoiceRespJob`，确认执行周期和告警。
- 确认 Global 使用的 BS SFTP 实际存在 `/DCS/SMMX/Response_Invoice/Pending/` 和 `Processed/`；如 BS 为 Global 分配了新目录，应同步修改常量。
- 使用脱敏样例分别验证 `Response_...` 和 `Response_Cancelled...` 文件格式及列顺序。
- 验证成功文件进入 `Processed`，失败文件保留在 `Pending` 且可通过 XXL-Job 日志定位。
- 开票回执后核对 `invoice_status=1`、`bs_invoice_status=3`、`sync_status=Get Invoice Success`。
- 取消回执后核对取消领域流程完整执行。

代码静态检查已通过；完整 Maven 编译受本地私有依赖缺失影响，尚未进入源码编译阶段。

## 取消发票 SFTP 交互流程

### 目录和责任方

```text
DCS 上传取消请求
    -> /DCS/SMMX/Invoice_Cancellation/Pending/

BLUE SERV 读取或受理请求
    -> /DCS/SMMX/Invoice_Cancellation/Processed/

BLUE SERV 生成取消回执
    -> /DCS/SMMX/Response_Invoice/Pending/

DCS 的 SynInvoiceRespJob 消费并归档回执
    -> /DCS/SMMX/Response_Invoice/Processed/
```

`Invoice_Cancellation/Pending` 到 `Processed` 的移动由 BLUE SERV 负责，Global 和 SMMX 后端均没有移动该请求文件的代码。BLUE SERV 的读取周期和“读取即移动”还是“处理成功后移动”需要由 BLUE SERV 团队确认。

### 文件约定

Global 当前取消请求文件名为：

```text
Can_<invoiceNo>_<yyyyMMdd>_<HHmmssSSS>.xls
```

取消回执由 `SynInvoiceRespJob` 按文件名前缀 `Response_Cancelled` 识别。联调流程图如果把取消请求标成 `Invoice#_...xls`，应改为实际的 `Can_...xls`。

### 联调检查点

- 开票请求仍在 `/INVOICE/Pending` 时，Cancel 应删除该请求并直接内部取消。
- 开票请求已被 BS 取走但尚无开票回执时，Cancel 应被阻止。
- `bs_invoice_status=INVOICED(3)` 时，Cancel 应生成 `Can_*.xls`，不能直接把发票改为取消。
- 发送取消请求后应为 `invoice_status=CREATED(1)`、`bs_invoice_status=CANCELLING(2)`。
- BS 取消回执处理后应为 `invoice_status=CANCELLED(0)`、`sync_status=Get Cancellation Success`。
- 当前代码会保留 `bs_invoice_status=CANCELLING(2)`；最终应清空还是新增 `CANCELLED` 状态，待业务确认。
