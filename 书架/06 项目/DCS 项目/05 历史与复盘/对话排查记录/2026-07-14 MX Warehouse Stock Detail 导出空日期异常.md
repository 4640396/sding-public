---
title: "MX Warehouse Stock Detail 导出空日期异常"
type: log
status: verified
created: 2026-07-14
updated: 2026-07-14
sensitivity: internal
source:
project: DCS
repositories:
  - dcs-global-vehicle
---
# MX Warehouse Stock Detail 导出空日期异常

## 现象

MX Warehouse Stock Detail 页面导出 Excel 时，global 后端抛出异常：

```text
java.lang.IllegalArgumentException: Date may not be null
org.apache.http.client.utils.DateUtils.formatDate(...)
MxWarehouseStockDetailViewController.lambda$downloadByVin(...)
```

第一次日志指向 `MxWarehouseStockDetailViewController.java:63` 的 ATA 格式化。只给 ATA 判空后再次出现相同异常，但后续日志没有具体代码行。

## 根因

导出逻辑直接使用 Apache HttpClient 的 `DateUtils.formatDate(Date)` 格式化日期：

```java
x.setAtaStr(DateUtils.formatDate(x.getAta()));
```

该方法不接受 `null`，任一导出记录的日期字段为空都会抛出 `IllegalArgumentException: Date may not be null`，并中断整个 Excel 导出。

这些日期字段并非本次新增。Git 历史显示，相关导出逻辑在 2026-06-02 的提交 `793f82aa49` 中已经存在。本次修改只是增加空值兼容。

第一次由 ATA 最先触发；ATA 判空后，循环会继续格式化 ATD、ETA、Arrived Date 等字段，因此其他空日期仍会触发相同异常。这也是仅修改 ATA 后问题再次出现的原因。

## ATA 数据来源

Warehouse Stock Detail 查询通过车辆 VIN 左连接 `lm_delivery_note`：

```sql
LEFT JOIN lm_delivery_note AS t7 ON t1.vin = t7.vin
    AND #{loginUserUuid} = t7.to_destination_org_uuid
    AND t7.data_status = 1
    AND t7.delivery_note_type = 1
```

ATA 来自：

```sql
t7.actual_delivery_time AS ata
```

`ATA` 表示 Actual Time of Arrival。以下情况会使其为空：

- 匹配到的 DN 尚未填写 `actual_delivery_time`；
- VIN 没有匹配到符合条件的 DN；
- DN 的目的组织、数据状态或类型不符合连接条件。

由于查询使用 `LEFT JOIN`，这些空值属于允许出现的业务数据，导出层需要兼容。

## 修改内容

目标文件：

```text
dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/warehouse/userinterface/representation/MxWarehouseStockDetailViewController.java
```

在 `downloadByVin()` 中为全部日期格式化增加空值判断：

```java
x.setAtaStr(x.getAta() == null ? null : DateUtils.formatDate(x.getAta()));
x.setAtdStr(x.getAtd() == null ? null : DateUtils.formatDate(x.getAtd()));
x.setEtaStr(x.getEta() == null ? null : DateUtils.formatDate(x.getEta()));
x.setArrivedDateStr(x.getArrivedDate() == null ? null : DateUtils.formatDate(x.getArrivedDate()));
x.setOfflineDateStr(x.getOfflineDate() == null ? null : DateUtils.formatDate(x.getOfflineDate()));
x.setRepuveDateStr(x.getRepuveDate() == null ? null : DateUtils.formatDate(x.getRepuveDate()));
x.setGateOutDateStr(x.getGateOutDate() == null ? null : DateUtils.formatDate(x.getGateOutDate()));
x.setGateInDateStr(x.getGateInDate() == null ? null : DateUtils.formatDate(x.getGateInDate()));
x.setDnDateStr(x.getDnDate() == null ? null : DateUtils.formatDate(x.getDnDate()));
```

处理后的行为：日期有值时保持原格式导出；日期为空时对应 Excel 单元格留空，不再中断整个导出。

## 额外注意

当前 `findNewWarehouseStockDetailViewPage` SQL 将 `t10.gate_in_received_time` 映射为 `arrivedDate`：

```sql
t10.gate_in_received_time AS arrivedDate
```

但导出代码同时读取 `getGateInDate()`。当前查询未直接映射 `gateInDate`，`queryNewPageList()` 也未给它赋值，因此该字段可能长期为空。此次判空可以避免导出异常，但 `Gate In Date` 是否应展示与 `Arrived Date` 相同或不同的数据，需要结合业务口径另行确认。

## 验证情况

- `git diff --check` 通过。
- Maven 编译未完成：项目依赖的内部制品 `com.smil:dcs-excel-bean:0.0.1` 和 SAP JCo `3.0.18` 无法从 Maven Central 获取，`dcs-global-api` 依赖解析失败；该失败与本次 Java 空值判断无关。
- 需要重新构建、部署 global 后端后，在包含空日期的 Warehouse Stock Detail 数据上重新验证导出。

## 结论

问题不是日期格式内容错误，而是原导出代码假设所有日期都有值。业务查询允许日期为空，但 Apache `DateUtils.formatDate()` 不接受空值。应对导出的全部可空日期统一判空，而不是只处理日志中第一次出现的 ATA。
