---
title: "03 数据库字段与 SQL 报错"
type: log
status: reference
created: 2026-07-07
updated: 2026-07-11
sensitivity: internal
source:
project: DCS
---
# 数据库字段与 SQL 报错

## MX Customer Goods Receive：pod_document 缺列

### 现象

接口：

```text
/mx/customerGoodsReceive/pagelist
http://172.29.10.11/api/dcs-smmx-veh-api/mx/customerGoodsReceive/pagelist
```

报错：

```text
Unknown column 'g.pod_document' in 'field list'
The error may exist in class path resource [mappers/lm/CustomerGoodsReceiveDaoMapper.xml]
```

SQL 中：

```sql
left join lm_customs_goods_receive g on b.vin = g.vin
select ..., g.notes, g.pod_document, ...
```

### 根因

代码已经使用 `lm_customs_goods_receive.pod_document`，但目标库表结构没有同步该字段。

PageHelper 分页会包一层：

```sql
select count(0) from (...) tmp_count
```

即便只是 count，MySQL 也会校验子查询里的字段，所以 count 阶段也会报 `Unknown column`。

### 修复

先确认字段：

```sql
SHOW COLUMNS FROM smil_dcs_global_vehicle.lm_customs_goods_receive LIKE 'pod_document';
```

缺字段则补：

```sql
ALTER TABLE smil_dcs_global_vehicle.lm_customs_goods_receive
ADD COLUMN pod_document varchar(1024) DEFAULT NULL COMMENT 'POD Document' AFTER notes;
```

### 潜在代码问题

`CustomerGoodsReceiveDao` 已有 `podDocument` 字段，但 `CustomerGoodsReceiveDao.of(...)` 没有把 domain 的 `podDocument` set 进去。

如果 MX 确认收车时要同步保存 POD，需要补：

```java
.podDocument(domain.getPodDocument())
```

### 结论

当前报错不是接口参数问题，是数据库结构落后于代码。

## SQL 对比：queryAllByVin

曾对比 smmx 和 global 的 `queryAllByVin` SQL。结论：主 SQL 逻辑完全一致。

一致项：

- SELECT 字段一致。
- FROM / JOIN 一致。
- WHERE `vein.business_status != 4` 一致。
- 动态条件一致：`vin1`、`vehicleMaterialCode1`、`customerName`、`customerCountry`、`warehouseNo`、`series`、`vehicleStatus`、`logisticsStatus`。
- 排序一致：`order by vein.create_time desc`。

唯一 SQL 文件层面的差异是 `resultType` 包名不同：

```xml
com.smil.smmxvehicle.warehouse.userinterface.vpo.InventoryPageByVinVPO
com.smil.globalvehicle.warehouse.userinterface.vpo.InventoryPageByVinVPO
```

如果两个 VPO 字段、setter、类型不同，Java 结果对象可能不同。SQL 查询结果本身应一致。







