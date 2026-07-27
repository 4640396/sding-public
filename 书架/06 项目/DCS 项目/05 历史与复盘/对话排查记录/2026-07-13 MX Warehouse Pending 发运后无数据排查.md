---
title: "MX Warehouse Pending 发运后无数据排查"
type: log
status: verified
created: 2026-07-13
updated: 2026-07-13
sensitivity: internal
source:
project: DCS
repositories:
  - dcs-global-vehicle
---
# MX Warehouse Pending 发运后无数据排查

## 现象

调用全球整车 GDS 发运接口后，MX Warehouse Pending 页面查询无结果：

```text
POST /gds/delivery
POST /asn/warehouse/pending/mx/pageListReform
```

Pending 列表请求条件：

```json
{
  "pendingStatus": "1",
  "page": {
    "pageNum": 1,
    "pageSize": 10
  }
}
```

GDS 发运数据已接收，但 `lm_asn_warehouse_pending` 和 `lm_asn_warehouse_pending_item` 没有对应 Pending 记录。

## 根因

UAT 环境缺少对应销售公司的入库模式配置，即 `lm_inbound_mode_configuration` 中没有对应 `sales_company_uuid` 且 `pending_needed = 1` 的记录。

`/gds/delivery` 并不是每次发运都创建 Warehouse Pending。只有同时满足以下条件才进入 Pending 池：

1. 客户号能找到对应组织；
2. 该组织在 `lm_inbound_mode_configuration` 中存在配置；
3. `pending_needed = 1`；
4. 订单目的地类型为 `WAREHOUSE`。

配置不存在时，`verifyEntryWarehousePending()` 直接返回 `false`，发运流程转而执行 `createDeliveryNotes(...)`，不会执行 `insertWarehousePending(...)`。

## 代码调用链

```text
POST /gds/delivery
→ LmGdsShippingController.delivery()
→ LmGDSDeliveryService.saveDeliveryInfo()
→ InboundModeConfigurationService.verifyEntryWarehousePending()
→ 满足 Pending 配置且 destinationType = WAREHOUSE
→ LmGDSDeliveryService.insertWarehousePending()
→ AsnWarehousePendingService.insertWarehousePending()
→ AsnWarehousePendingRepositoryImpl.save()
```

关键分支：

```java
if (enterThePendingPool && destinationType == DestinationTypeEnum.WAREHOUSE) {
    this.insertWarehousePending(deliveryInterfaceDao, lmOrders);
} else {
    this.createDeliveryNotes(deliveryInterfaceDao, lmOrders, new HashMap<>(), warehouse, false);
}
```

## 数据表关系

| 数据表 | 作用 |
| --- | --- |
| `lm_inbound_mode_configuration` | 控制指定销售公司是否进入 Warehouse Pending 池 |
| `lm_gds_delivery_interface` | GDS 发运接口主表 |
| `lm_gds_delivery_interface_item` | GDS 发运接口明细，包含 `vehicle_vin` 和 `sys_vin` |
| `lm_asn_warehouse_pending` | Pending 主表，不存 VIN |
| `lm_asn_warehouse_pending_item` | Pending 明细表，VIN 存在此表 |

## VIN 字段注意点

写入 Pending 明细时，当前代码使用 GDS 明细的 `systemVin`，对应数据库字段 `sys_vin`，不是 `vehicleVin`：

```java
.map(item -> new ASNWarehouseVDO(item.getSystemVin(), item.getMaterialId()))
```

因此排查 VIN 时需要同时查 `vehicle_vin` 和 `sys_vin`。另外，MX Pending 列表 SQL 当前使用：

```sql
lm_asn_warehouse_pending_item.vin = lm_gds_delivery_interface_item.vehicle_vin
```

如果 `sys_vin` 与 `vehicle_vin` 不相同，可能出现 Pending 已写入，但列表被内连接过滤的情况。这是独立的潜在问题，不是本次未写入 Pending 的直接根因。

## 排查 SQL

检查是否存在 Pending 配置：

```sql
SELECT configuration_id,
       sales_company_uuid,
       pending_needed
FROM lm_inbound_mode_configuration
WHERE sales_company_uuid = :salesCompanyUuid;
```

期望结果是存在一条 `pending_needed = 1` 的记录。

检查 GDS 接收的两类 VIN：

```sql
SELECT h.invoice,
       h.customer_number,
       i.vehicle_vin,
       i.sys_vin,
       i.customer_order_no
FROM lm_gds_delivery_interface_item i
JOIN lm_gds_delivery_interface h
  ON h.id = i.lm_delivery_interface_id
WHERE i.vehicle_vin = :vin
   OR i.sys_vin = :vin;
```

检查 Pending 主表和明细表：

```sql
SELECT a.pending_id,
       a.document_no,
       a.data_status AS main_status,
       b.vin,
       b.data_status AS item_status
FROM lm_asn_warehouse_pending a
LEFT JOIN lm_asn_warehouse_pending_item b
  ON b.pending_id = a.pending_id
WHERE a.document_no = :invoice
   OR b.vin = :systemVin;
```

## 处理与验证建议

1. 通过环境初始化脚本或受控配置流程，为目标销售公司补齐 `pending_needed = 1` 的配置。
2. 补配置后使用新的发票号和 VIN 重新发运验证；已成功接收的原发运数据可能受发票号、VIN 防重校验限制，不应直接重放。
3. 确认新数据写入 `lm_asn_warehouse_pending` 和 `lm_asn_warehouse_pending_item`。
4. 调用 `/asn/warehouse/pending/mx/pageListReform` 确认 `pendingStatus = 1` 能查到数据。
5. 若表中有 Pending 但页面仍无数据，继续核对 `sys_vin` 与 `vehicle_vin` 的关联问题。

## 结论

本次问题的直接原因是 UAT 缺少销售公司的 Warehouse Pending 入库模式配置，导致 `/gds/delivery` 走直接创建 DN 的分支，未调用 Pending 写入逻辑。

## 关键代码

```text
dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/logistics/userinterface/restful/LmGdsShippingController.java
dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/logistics/infrastructure/adapter/service/LmGDSDeliveryService.java
dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/logistics/application/service/InboundModeConfigurationService.java
dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/logistics/application/service/AsnWarehousePendingService.java
dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/logistics/infrastructure/repository/AsnWarehousePendingRepositoryImpl.java
dcs-global-vehicle/dcs-service/src/main/resources/mappers/lm/AsnWarehousePendingMapper.xml
```
