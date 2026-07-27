---
title: UAT SMP订单重导清理发票与Pending残留
type: log
status: draft
created: 2026-07-22
updated: 2026-07-25
sensitivity: internal
project: DCS
repositories:
  - dcs-global-vehicle
sources:
  - "[[2026-07-13 MX Warehouse Pending 发运后无数据排查]]"
  - "[[../../03 实施与验收/墨西哥OEM库存与历史订单迁移/00 执行总览]]"
  - "dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/tradeorder/userinterface/ops/OpsSmpOrderDataInitializationController.java"
  - "dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/tradeorder/application/service/initial/OpsSmpDataInitService.java"
  - "dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/logistics/infrastructure/adapter/service/LmGDSDeliveryService.java"
  - "dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/logistics/infrastructure/adapter/service/LmDeliveryService.java"
  - "dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/logistics/infrastructure/adapter/mappers/LmDeliveryInterfaceMapper.java"
---

# UAT SMP订单重导清理发票与Pending残留

## 背景

UAT 通过 `POST /api/dcs-global-api/ops/smpOrder/save` 重新导入 SMP 采购与发运数据时，批处理顶层返回成功，但订单进入 `failed`：

```text
lm.delivery.invoice.exist
```

用户此前已经删除目标范围的 `tom_trade_order` 与 `tom_vehicle_order`，但首次导入生成的物流接口和 Warehouse Pending 数据仍然保留，导致重导被发票防重校验拦截。

## 已核实的代码事实

`OpsSmpDataInitService.save()` 先保存订单，再将发运数据交给 `LmGDSDeliveryService.saveDeliveryInfo()`。

发运服务首先调用：

```java
deliveryService.hasDocumentNo(deliveryInterfaceDao.getInvoice())
```

底层查询条件是：

```sql
SELECT *
FROM lm_gds_delivery_interface
WHERE invoice_no = :invoiceNo
  AND e_type = 'S'
LIMIT 1;
```

因此，只删除订单主数据并不能解除发票防重。只要成功状态的 GDS 接口主记录仍在，重导仍会报 `lm.delivery.invoice.exist`。

当入库模式要求进入 Warehouse Pending 且订单目的地为 `WAREHOUSE` 时，发运流程不会创建正式 DN，而是写入：

- `lm_asn_warehouse_pending`
- `lm_asn_warehouse_pending_item`

否则才会写入 `lm_delivery_note` 和 `lm_delivery_note_gds_info`。

## 本次数据证据

本次重导范围共 73 张订单。删除前只读统计为：

| 数据 | 数量 |
|---|---:|
| GDS 接口主记录 | 148 |
| GDS 接口明细 | 6021 |
| ASN Pending 主记录 | 148 |
| ASN Pending 明细 | 6021 |

两项安全检查由用户确认均为 `0`：

```text
shared_outside_order_count = 0
formal_delivery_note_count = 0
```

这表示目标发票未与清单外订单共用，且没有生成正式发运单；本次清理范围可限定在 ASN Pending 与 GDS 接口四张表。

## 清理顺序

由于订单主数据此前已经删除，本次未再次修改订单表。清理顺序为：

```text
lm_asn_warehouse_pending_item
→ lm_asn_warehouse_pending
→ lm_gds_delivery_interface_item
→ lm_gds_delivery_interface
```

当前人工执行脚本位于：

```text
$env:WORKS_ROOT\dcs\tools\uat_smp_reimport_cleanup\cleanup_pending_gds_no_temp_table.sql
```

脚本使用订单号会话变量反查接口 ID 和发票号，不在 Vault 保留独立 SQL 副本。

## 执行中遇到的问题

### 无临时表权限

数据库账号执行 `CREATE TEMPORARY TABLE` 时返回：

```text
Error Code: 1044. Access denied
```

处理方式是改用 `@order_nos` 会话变量和 `DELETE JOIN`，不再创建临时表。断开或切换数据库连接后必须重新设置会话变量。

### Safe update 拦截

Workbench 对 `DELETE JOIN` 返回：

```text
Error Code: 1175. You are using safe update mode
```

处理方式是在当前会话暂时保存并关闭 safe update，事务结束后恢复：

```sql
SET @old_sql_safe_updates = @@SESSION.sql_safe_updates;
SET SESSION sql_safe_updates = 0;
```

提交或回滚后执行：

```sql
SET SESSION sql_safe_updates = @old_sql_safe_updates;
```

### 大 JOIN 导致连接中断

初版删除将目标接口明细与 Pending 明细直接关联，造成中间结果重复放大，执行时出现：

```text
Error Code: 2013. Lost connection to MySQL server during query
```

连接中断后未提交事务应由数据库回滚，但必须重新连接并复查删除前数量，不能只依赖推断。后续脚本先按 `invoice_no` 或 `lm_delivery_interface_id` 分组去重，再执行删除。

## 可复用排查路线

遇到 `lm.delivery.invoice.exist` 时：

1. 不要把顶层 `success=true` 当成订单导入成功，检查 `successList`、`duplicateIgnored` 和 `failed`。
2. 从请求中的订单号查询 `lm_gds_delivery_interface_item.customer_order_no`。
3. 关联 `lm_gds_delivery_interface`，确认 `invoice_no`、`e_type`、创建时间和明细数量。
4. 查询 `lm_delivery_note_gds_info`，判断是否已经形成正式 DN。
5. 查询 `lm_asn_warehouse_pending.document_no = lm_gds_delivery_interface.invoice_no`，判断是否进入 Pending 池。
6. 批量删除前检查同一接口是否包含清单外订单，避免误删共用发票。
7. 重导清理不能只删 Pending；还必须删除对应 GDS 接口数据，否则仍会触发发票或 VIN 防重。
8. 所有删除放在同一事务中，核对影响行数和剩余数量后再提交。

## 当前状态与未验收项

用户已反馈清理 SQL 执行完成，但当前对话没有取得以下最终证据：

- 三段删除语句的最终影响行数；
- `COMMIT` 后四张表剩余数量为零的查询结果；
- 重新调用 `/ops/smpOrder/save` 后 73 张订单的 `successList`、`duplicateIgnored` 与 `failed`；
- 重导后订单、接口、Pending 和 VIN 数量是否与源模板一致。

因此本记录保持 `status: draft`。取得上述证据后再更新为 verified，且不得将 UAT 结果表述为生产就绪。

## 2026-07-24 B01–B14 全量回退范围补充

### 背景和范围

V2 期初库存模板已经补齐为 B01–B14，共覆盖 18,449 个业务 VIN。若 UAT 导入结果错误，清理范围应以这 18,449 个 VIN 为主键集合，并由 VIN 固化反查采购订单号、Vehicle Order、DN、GDS 接口 ID、Pending ID 和库存编号。`B01`–`B14` 只是文件批次标识，不保证落入数据库字段，因此不能直接作为删除条件。

本节记录的是依据当前 `dcs-global-vehicle` 源码核对得到的落表链路和人工 SQL 设计边界，不代表已经在 UAT 执行或验收。

### 已核对的保存链路

`POST /ops/smpOrder/save` 对每张采购订单分别调用初始化服务；单张订单事务内依次执行：

1. 保存采购订单聚合；
2. 初始化订单物料数量状态；
3. 创建 Vehicle Order 及物流信息；
4. 保存 GDS 发运接口主表和明细；
5. 根据入库模式进入 Warehouse Pending，或创建正式 DN；
6. 发运事件创建或更新车辆实例，并更新 Vehicle Order 的 VIN、DN、价格和业务状态。

Gate In 通过独立入口执行，因此库存和 Gate In 表属于条件清理范围，不能假定每次采购发运导入都已写入。

### 必查表

采购订单及 Vehicle Order：

- `tom_trade_order`
- `tom_trade_order_info`
- `tom_trade_order_snapshot`
- `tom_trade_order_material`
- `tom_trade_order_stat`
- `tom_vehicle_order`
- `tom_vehicle_order_logistics_info`

GDS 发运及正式 DN：

- `lm_gds_delivery_interface`
- `lm_gds_delivery_interface_item`
- `lm_delivery_note`
- `lm_delivery_note_gds_info`
- `lm_delivery_note_info`，仅在实际存在对应 DN 信息时清理

车辆实例：

- `vehicle_instance`

### 条件触发表

进入 Warehouse Pending 时：

- `lm_asn_warehouse_pending`
- `lm_asn_warehouse_pending_item`

已经执行 Gate In 时：

- `warehouse_inventory_vehicle`
- `warehouse_inventory_vehicle_gated_in_note`
- `warehouse_inventory_vehicle_gated_out_note`

`warehouse_inventory_vehicle_gated_out_note` 正常不应由期初库存 Gate In 产生。只要命中 Gate Out、销售、发票、结算、凭证、转移或其他下游业务，必须停止通用物理删除，转为专项回退评估。

### 推荐清理顺序

```text
先固化 VIN、Vehicle Order、采购订单、DN、接口 ID、Pending ID、库存编号
→ Gate In / Gate Out 明细
→ 库存车辆
→ Pending 明细
→ 无剩余明细的 Pending 主表
→ DN GDS 信息与 DN 信息
→ DN 主表
→ GDS 接口明细
→ 无剩余明细的 GDS 接口主表
→ Vehicle Order 物流信息
→ Vehicle Order
→ 订单状态、物料、快照、信息
→ 采购订单主表
```

删除主表前必须再次检查是否仍有本次范围外明细：

- 同一个 GDS 接口仍有范围外 VIN 时，只删目标接口明细，不删接口主表；
- 同一个 Pending 仍有范围外 VIN 时，只删目标 Pending 明细，不删 Pending 主表；
- 同一个采购订单仍有范围外 Vehicle Order 时，不得整单删除采购订单聚合。

### `vehicle_instance` 特殊风险

GDS 发运监听器对 `vehicle_instance` 既可能新增，也可能更新已有车辆。没有迁移前快照时，不能把所有目标 VIN 对应的 `vehicle_instance` 直接删除：

- 确认迁移前不存在的 VIN，才可作为候选删除；
- 迁移前已经存在的 VIN，需要按备份恢复原字段；
- 无法区分新增与更新时，默认保留并人工核对。

### 人工 SQL 使用规则

清理 SQL 应先建立或等效固化目标 VIN 范围，再执行只读统计和下游拦截检查。实际删除要求：

1. 仅在 UAT 使用；
2. 同一数据库连接内执行；
3. 所有删除放在一个事务内；
4. 首次演练保留 `ROLLBACK`；
5. 记录每张表删除前数量、影响行数和删除后剩余数；
6. 确认结果后才将 `ROLLBACK` 改为 `COMMIT`；
7. 正式清理前备份所有命中记录。

已有 UAT 账号曾出现无临时表权限和 safe update 拦截。若继续使用临时范围表，应先验证 `CREATE TEMPORARY TABLE` 权限；无权限时改用受控范围表或会话变量方案，不得为了绕过权限而放宽删除条件。

### 本次新增未验收项

- 尚未取得 B01–B14 在 UAT 的实际导入批次、成功订单数和失败订单数；
- 尚未执行 18,449 VIN 在各候选表的命中数量审计；
- 尚未确认是否已经执行 Gate In，以及是否存在 Gate Out 或其他下游业务；
- 尚未确认目标环境的实际表结构、外键、触发器和部署版本与当前源码完全一致；
- 通用清理骨架尚未在空库、回滚演练和重复执行场景验证；
- `vehicle_instance` 的新增与覆盖更新范围尚未取得迁移前快照证据。

在上述证据补齐前，本节只能作为 UAT 清理设计草案，不能作为生产清理脚本或已验收方案。
