---
title: Seller生产取数与SQL排查记录
type: history
status: deprecated
superseded_by: "[[../03 销售订单与发票/01 Seller模板生产只读取数]]"
created: 2026-07-18
updated: 2026-07-18
sensitivity: internal
project: DCS
sources:
  - C:/works/initdata/Order/经销商采购导入模版.xlsx
  - "[[../01 期初库存入库/02 批量生产只读取数]]"
  - "[[02 Seller批量执行方案]]"
  - dcs-global-vehicle/dcs-test/SQL/develop_global_vehicle_junit.sql
  - dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/tradeorder/application/service/initial/OpsSmzaDataInitService.java
  - dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/tradeorder/application/service/initial/vo/SmzaOrderVo.java
  - dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/tradeorder/application/service/initial/OpsDataInitInvoice.java
---

# Seller生产取数与SQL排查记录

> [!warning] 历史记录，不执行
> 本页保留 2026-07-18 的错误关联、状态诊断、0号验证和旧批次 SQL，仅用于追溯。当前运维只执行 [[../03 销售订单与发票/01 Seller模板生产只读取数]] 中的三条导出 SQL。

> [!important] 执行边界
> 本页 SQL 只允许在生产只读连接执行，不创建临时表、不写数据。模板原有南非数据只是参考数据，必须全部清除。三个导出结果分别粘贴到 `Order`、`Invoice`、`Logistics` Sheet，仅粘贴数据行，不覆盖表头。

> [!note] 全量口径
> 1、2、3 号 SQL 已改为直接从生产有效 Seller 订单全量导出，不传 VIN、不传 OEM 订单号、不加时间范围。这里的 `Order` 是墨西哥公司向 Dealer 销售的 Seller 订单，不是总部向墨西哥销售形成的 OEM 采购订单。正式装填前先执行 0 号全量候选 SQL，只保留 `scope_check=PASS` 的订单；再在本地与业务库存 VIN 清单做交集/冲突检查，禁止把 Excel 写入生产临时表。

> [!tip] 当前全量方案只执行四段 SQL
> `0号候选 → 1号Order → 2号Invoice → 3号Logistics → 本地验收`。正文中的四段 SQL 均不传 VIN、不传 OEM 订单号、不加时间范围；历史批次 SQL 已折叠到附录，正常执行时不要展开。

> [!warning] 2026-07-18 关联键修正
> 首版 0 号 SQL 错用 `lm_delivery_note.vehicle_order_no = tom_vehicle_order.vehicle_order_no`，并把有效发票限定为 `invoice_status=1`，生产实测造成 DN、Gate Out、Received、Invoice 计数全部为 0。SMMX 当前 Mapper 的真实关系是 `tom_vehicle_order.dn_no = lm_delivery_note.delivery_note_no`，发票明细按 `vehicle_order_no` 关联，有效状态按 `invoice_status>=1`。正文 0、2、3 号及历史验收 SQL 已统一修正；首版结果作废。

> [!important] 2026-07-18 生产状态分布实测
> `seller_order` 为生产 `L000`、Seller、已审批时：车辆状态 5 有 19 辆、状态 7 有 1,453 辆，均无有效 DN 和发票；车辆状态 8 有 218,480 辆且均有关联发票，其中 187,342 辆有 DN，186,108 辆同时有 Gate Out 和 Received。历史 Seller 模板候选因此固定为 `seller_vo.data_status=8`；缺 DN 或物流未完成的数据继续由 0 号 SQL 标记 STOP，不进入模板。上述数量是 2026-07-18 的生产查询证据，不是最终迁移数量，最终仍以整单 `scope_check=PASS` 为准。

> [!success] 0号 SQL 生产只读验证通过
> 2026-07-18 在生产只读查询平台重新执行修正版 0 号 SQL，返回记录的 `vehicle_order_status=8`，且已观察到 `seller_vin_count=dn_vin_count=gate_out_vin_count=received_vin_count=invoice_vin_count=1`、`scope_check=PASS`。这证明状态筛选、DN 关联和发票关联修正已生效。该截图仅证明存在合格候选，不代表全量 PASS 数量已经统计或模板已经验收。

当前期初库存 VIN 未找到 Seller 销售订单时，不代表 SQL 失败。未售库存本来就不应生成 Dealer 销售单或发票。Seller 阶段必须先从生产真实 `L000` 销售订单选择候选，再通过 VIN 追溯其上游 OEM 订单；不得把所有 Gate In 库存默认带入 Seller 阶段。

## 0、全量Seller销售订单候选

先执行本段，不传当前库存订单号。只有 `scope_check=PASS` 的完整 Seller 销售订单才能进入后续模板。结果中的上游 OEM 订单号用于追踪和核对；全量执行 1、2、3 号 SQL 时不再作为查询参数。

```sql
SELECT DISTINCT
    seller_order.trade_order_no AS seller_order_no,
    seller_order.customer_no AS dealer_sap_no,
    seller_order.delivery_destination AS delivery_short_name,
    GROUP_CONCAT(DISTINCT oem_order.trade_order_no
        ORDER BY oem_order.trade_order_no SEPARATOR ',') AS upstream_oem_order_nos,
    COUNT(DISTINCT seller_vo.vin) AS seller_vin_count,
    MAX(seller_vo.data_status) AS vehicle_order_status,
    COUNT(DISTINCT CASE WHEN dn.data_status = 1 THEN dn.vin END) AS dn_vin_count,
    COUNT(DISTINCT CASE WHEN dni.gate_out_time IS NOT NULL THEN dn.vin END) AS gate_out_vin_count,
    COUNT(DISTINCT CASE WHEN dni.gate_in_received_time IS NOT NULL THEN dn.vin END) AS received_vin_count,
    COUNT(DISTINCT CASE WHEN inv.invoice_status >= 1 THEN invoice_item.vin END) AS invoice_vin_count,
    CASE
      WHEN COUNT(DISTINCT seller_vo.vin) = 0
        THEN 'STOP:NO_VIN'
      WHEN COUNT(DISTINCT oem_order.trade_order_no) = 0
        THEN 'STOP:NO_UPSTREAM_OEM_ORDER'
      WHEN MAX(addr.id) IS NULL
        THEN 'STOP:DELIVERY_ADDRESS'
      WHEN COUNT(DISTINCT seller_vo.vin)
             <> COUNT(DISTINCT CASE WHEN dn.data_status = 1 THEN dn.vin END)
        THEN 'STOP:DN_NOT_COMPLETE'
      WHEN COUNT(DISTINCT seller_vo.vin)
             <> COUNT(DISTINCT CASE WHEN dni.gate_out_time IS NOT NULL THEN dn.vin END)
        THEN 'STOP:GATE_OUT_NOT_COMPLETE'
      WHEN COUNT(DISTINCT seller_vo.vin)
             <> COUNT(DISTINCT CASE WHEN dni.gate_in_received_time IS NOT NULL THEN dn.vin END)
        THEN 'STOP:RECEIVED_NOT_COMPLETE'
      WHEN COUNT(DISTINCT seller_vo.vin)
             <> COUNT(DISTINCT CASE WHEN inv.invoice_status >= 1 THEN invoice_item.vin END)
        THEN 'STOP:INVOICE_NOT_COMPLETE'
      ELSE 'PASS'
    END AS scope_check
FROM tom_trade_order seller_order
JOIN tom_vehicle_order seller_vo
  ON seller_vo.trade_order_no = seller_order.trade_order_no
LEFT JOIN tom_vehicle_order oem_vo
  ON oem_vo.vin = seller_vo.vin
LEFT JOIN tom_trade_order oem_order
  ON oem_order.trade_order_no = oem_vo.trade_order_no
 AND oem_order.sales_company_no = '5000'
 AND oem_order.customer_no = '270165'
 AND oem_order.data_status = 4
LEFT JOIN md_organization_customer_delivery_address addr
  ON addr.customer_no = seller_order.customer_no
 AND addr.delivery_short_name = seller_order.delivery_destination
LEFT JOIN lm_delivery_note dn
  ON dn.delivery_note_no = seller_vo.dn_no
 AND dn.data_status = 1
LEFT JOIN lm_delivery_note_info dni
  ON dni.delivery_note_no = dn.delivery_note_no
LEFT JOIN ic_invoice_item invoice_item
  ON invoice_item.vehicle_order_no = seller_vo.vehicle_order_no
LEFT JOIN ic_invoice inv
  ON inv.invoice_no = invoice_item.invoice_no
 AND inv.invoice_status >= 1
WHERE seller_order.sales_company_no = 'L000'
  AND seller_order.shipper = 1
  AND seller_order.data_status = 4
  AND seller_vo.data_status = 8
GROUP BY
    seller_order.trade_order_no,
    seller_order.customer_no,
    seller_order.delivery_destination
ORDER BY scope_check, seller_order.trade_order_no;
```

候选结果可能较多，可先由生产查询平台限制返回行数，但不得通过随意增加状态条件隐藏 `STOP`。试跑时选一个 `PASS` Seller 订单核对；正式全量导出时按 1、2、3 号 SQL 直接查询。

## 固定口径与环境映射

`<BATCH_OEM_ORDER_NO_LIST>` 仅保留给折叠的历史排查附录使用；全量导出模板的 0、1、2、3 号 SQL 没有任何占位符，可以直接执行。需要执行旧批次排查时，再把占位符替换为 [[../01 期初库存入库/02 批量生产只读取数]] 当前批次的生产 OEM 采购订单号列表，例如：

```text
'OEM_ORDER_001','OEM_ORDER_002'
```

当前单单验证如果使用之前的 OEM 订单 `MXASMIL2412150006`，则 SQL 必须实际改成：

```sql
WHERE oem_vo.trade_order_no IN ('MXASMIL2412150006')
```

不能直接执行仍包含 `<BATCH_OEM_ORDER_NO_LIST>` 的旧批次 SQL；尖括号占位符不是合法 SQL。执行全量模板导出时无需替换，也不要人为追加发票日期范围。

```text
生产 OEM 卖方销售公司：5000
生产墨西哥组织客户号：270165
生产墨西哥 Seller 销售公司：L000
UAT 墨西哥 Seller 销售公司：E000
Seller 类型：shipper = 1
已审批订单：data_status = 4
```

取数关系：

```text
批次 OEM 采购订单
  -> tom_vehicle_order 取得批次 VIN
  -> 同 VIN 的 L000 Seller 销售订单
  -> Dealer、目的地、物料、售价
  -> DN、Gate Out、Received
  -> Seller 发票
```

只有 `Sales Company No` 做环境映射：生产 `L000` 输出为 UAT `E000`。订单号、Dealer、VIN、物料、价格、日期和发票号不得人工改造。

<details>
<summary>历史附录 A：旧批次逐 VIN 完整性预检（正常流程不执行）</summary>

## 历史批次逐VIN完整性预检

本段只在需要按 `<BATCH_OEM_ORDER_NO_LIST>` 排查某批 OEM 订单时执行，不是全量流程的第二个“0号 SQL”。一辆 VIN 必须只对应一个目标 Seller 销售订单、一个有效 DN 和一个有效 Seller 发票；`scope_check` 必须全部为 `PASS`。

2026-07-18 生产查询平台的权限解析器先后无法解析 `LEFT JOIN ... ON ... EXISTS (...)` 和 `LEFT JOIN (SELECT ...)`，均报 `Required keyword: 'this' missing`。下方最终修订为只连接实体表的平铺 SQL，不再使用 `EXISTS` 或 JOIN 派生表；这是查询语法兼容修订，业务筛选条件未改变。

```sql
SELECT
    oem_vo.vin,
    COUNT(DISTINCT seller_order.trade_order_no) AS seller_order_count,
    GROUP_CONCAT(DISTINCT seller_order.trade_order_no
        ORDER BY seller_order.trade_order_no SEPARATOR ',') AS seller_order_nos,
    COUNT(DISTINCT CASE WHEN dn.data_status = 1 THEN dn.delivery_note_no END) AS active_dn_count,
    COUNT(DISTINCT CASE WHEN inv.invoice_status >= 1 THEN inv.invoice_no END) AS active_invoice_count,
    MAX(CASE WHEN seller_order.trade_order_no IS NOT NULL THEN seller_order.customer_no END) AS dealer_sap_no,
    MAX(CASE WHEN seller_order.trade_order_no IS NOT NULL THEN seller_order.delivery_destination END) AS delivery_short_name,
    MAX(CASE WHEN seller_order.trade_order_no IS NOT NULL THEN seller_vo.process_warehouse END) AS from_warehouse,
    MAX(CASE WHEN seller_order.trade_order_no IS NOT NULL THEN DATE(seller_vo.dn_time) END) AS dn_date,
    MAX(DATE(dni.gate_out_time)) AS gate_out_date,
    MAX(DATE(dni.gate_in_received_time)) AS received_date,
    CASE
      WHEN COUNT(DISTINCT seller_order.trade_order_no) <> 1
        THEN 'STOP:SELLER_ORDER_COUNT'
      WHEN COUNT(DISTINCT CASE WHEN dn.data_status = 1 THEN dn.delivery_note_no END) <> 1
        THEN 'STOP:DN_COUNT'
      WHEN COUNT(DISTINCT CASE WHEN inv.invoice_status >= 1 THEN inv.invoice_no END) <> 1
        THEN 'STOP:INVOICE_COUNT'
      WHEN MAX(addr.id) IS NULL
        THEN 'STOP:DELIVERY_ADDRESS'
      WHEN MAX(CASE WHEN seller_order.trade_order_no IS NOT NULL THEN seller_vo.process_warehouse END) IS NULL
        OR MAX(CASE WHEN seller_order.trade_order_no IS NOT NULL THEN seller_vo.process_warehouse END) = ''
        THEN 'STOP:WAREHOUSE'
      WHEN MAX(CASE WHEN seller_order.trade_order_no IS NOT NULL THEN seller_vo.dn_time END) IS NULL
        THEN 'STOP:DN_DATE'
      WHEN MAX(dni.gate_out_time) IS NULL
        THEN 'STOP:GATE_OUT_DATE'
      WHEN MAX(dni.gate_in_received_time) IS NULL
        THEN 'STOP:RECEIVED_DATE'
      ELSE 'PASS'
    END AS scope_check
FROM tom_vehicle_order oem_vo
LEFT JOIN tom_vehicle_order seller_vo
  ON seller_vo.vin = oem_vo.vin
LEFT JOIN tom_trade_order seller_order
  ON seller_order.trade_order_no = seller_vo.trade_order_no
 AND seller_order.sales_company_no = 'L000'
 AND seller_order.shipper = 1
 AND seller_order.data_status = 4
LEFT JOIN md_organization_customer_delivery_address addr
  ON addr.customer_no = seller_order.customer_no
 AND addr.delivery_short_name = seller_order.delivery_destination
LEFT JOIN lm_delivery_note dn
  ON dn.delivery_note_no = seller_vo.dn_no
 AND seller_order.trade_order_no IS NOT NULL
 AND dn.data_status = 1
LEFT JOIN lm_delivery_note_info dni
  ON dni.delivery_note_no = dn.delivery_note_no
LEFT JOIN ic_invoice_item invoice_item
  ON invoice_item.vehicle_order_no = seller_vo.vehicle_order_no
LEFT JOIN ic_invoice inv
 ON inv.invoice_no = invoice_item.invoice_no
 AND inv.invoice_status >= 1
WHERE oem_vo.trade_order_no IN (<BATCH_OEM_ORDER_NO_LIST>)
  AND oem_vo.vin IS NOT NULL
GROUP BY oem_vo.vin
ORDER BY oem_vo.vin;
```

若出现 `STOP`，不得进入模板导出。特别注意：没有 Seller 订单，通常表示该 OEM 库存车尚未真实销售，不应伪造销售单、物流日期或发票。

### 0A. STOP:SELLER_ORDER_COUNT诊断

若 `seller_order_count=0`，按 VIN 执行以下只读 SQL，列出生产库中该车关联的全部订单。它只用于判断是尚未销售，还是销售公司、shipper 或状态与预期不同；不得据此放宽条件后直接导入。

```sql
SELECT
    vo.vin,
    vo.trade_order_no,
    tor.sales_company_no,
    tor.customer_no,
    tor.shipper,
    tor.data_status AS trade_order_status,
    tor.order_type,
    tor.source_code,
    tor.pricing_date,
    vo.data_status AS vehicle_order_status,
    vo.process_warehouse,
    vo.dn_no,
    vo.dn_time,
    vo.invoice_no,
    vo.invoice_time
FROM tom_vehicle_order vo
JOIN tom_trade_order tor
  ON tor.trade_order_no = vo.trade_order_no
WHERE vo.vin = '<TARGET_VIN>'
ORDER BY tor.create_time, vo.trade_order_no;
```

当前结果对应的 VIN 可替换为：

```sql
WHERE vo.vin = 'LSJA36E97PZ205664'
```

</details>

## 1、全量导出Order Sheet

结果列顺序与模板 `Order` Sheet 完全一致，一辆 VIN 一行。

```sql
SELECT DISTINCT
    seller_order.trade_order_no                       AS `Order No`,
    'E000'                                            AS `Sales Company No`,
    seller_order.incoterm_code                        AS `Incoterm Code`,
    DATE_FORMAT(seller_order.pricing_date, '%Y-%m-%d') AS `Order Date`,
    seller_order.delivery_destination                 AS `DELIVERY SHORT NAME`,
    seller_order.customer_no                          AS `DEALER SAP NO`,
    seller_order.delivery_destination                 AS `DELIVERY SHORT NAME`,
    addr.name                                         AS `Delivery Name`,
    addr.country_code                                 AS `Country Code`,
    addr.address                                      AS `ADDRESS`,
    seller_vo.vin                                     AS `VIN`,
    seller_order.order_type                           AS `Order Type`,
    material.unit_price                               AS `Sales Price`,
    seller_order.currency                             AS `Currency`,
    seller_vo.vehicle_material_code                   AS `Material Code`,
    material.material_description                     AS `Material Description`,
    material.interior_description                     AS `Interior Description`,
    material.exterior_description                     AS `Exterior Description`
FROM tom_trade_order seller_order
JOIN tom_vehicle_order seller_vo
  ON seller_vo.trade_order_no = seller_order.trade_order_no
 AND seller_vo.vin IS NOT NULL
 AND seller_vo.vin <> ''
JOIN tom_trade_order_material material
  ON material.trade_order_no = seller_order.trade_order_no
 AND material.vehicle_material_code = seller_vo.vehicle_material_code
JOIN md_organization_customer_delivery_address addr
  ON addr.customer_no = seller_order.customer_no
 AND addr.delivery_short_name = seller_order.delivery_destination
JOIN lm_delivery_note dn
  ON dn.delivery_note_no = seller_vo.dn_no
 AND dn.data_status = 1
JOIN lm_delivery_note_info dni
  ON dni.delivery_note_no = dn.delivery_note_no
JOIN ic_invoice_item invoice_item
  ON invoice_item.vehicle_order_no = seller_vo.vehicle_order_no
JOIN ic_invoice inv
  ON inv.invoice_no = invoice_item.invoice_no
 AND inv.invoice_status >= 1
JOIN tom_vehicle_order oem_vo
  ON oem_vo.vin = seller_vo.vin
JOIN tom_trade_order oem_order
  ON oem_order.trade_order_no = oem_vo.trade_order_no
 AND oem_order.sales_company_no = '5000'
 AND oem_order.customer_no = '270165'
 AND oem_order.data_status = 4
WHERE seller_order.sales_company_no = 'L000'
 AND seller_order.shipper = 1
 AND seller_order.data_status = 4
 AND seller_vo.data_status = 8
 AND seller_vo.dn_time IS NOT NULL
 AND seller_vo.process_warehouse IS NOT NULL
 AND seller_vo.process_warehouse <> ''
 AND dni.gate_out_time IS NOT NULL
 AND dni.gate_in_received_time IS NOT NULL
 AND inv.invoice_date IS NOT NULL
 AND material.unit_price IS NOT NULL
ORDER BY seller_order.trade_order_no, seller_vo.vin;
```

说明：模板有两个同名 `DELIVERY SHORT NAME` 列，SQL 按模板原顺序输出两次相同的生产真实值。`E000` 是唯一环境映射；源订单必须由查询条件确认来自生产 `L000`。

## 2、全量导出Invoice Sheet

结果列顺序与模板 `Invoice` Sheet 一致，一辆 VIN 一行。

```sql
SELECT DISTINCT
    seller_order.trade_order_no                         AS `Order No`,
    seller_vo.vin                                       AS `VIN`,
    COALESCE(NULLIF(inv.biz_invoice_no, ''), inv.invoice_no) AS `Invoice No`,
    DATE_FORMAT(inv.invoice_date, '%Y-%m-%d')           AS `Invoice Date`,
    invoice_item.price                                  AS `Invoice Price`
FROM tom_trade_order seller_order
JOIN tom_vehicle_order seller_vo
  ON seller_vo.trade_order_no = seller_order.trade_order_no
 AND seller_vo.vin IS NOT NULL
 AND seller_vo.vin <> ''
JOIN ic_invoice_item invoice_item
  ON invoice_item.vehicle_order_no = seller_vo.vehicle_order_no
JOIN ic_invoice inv
  ON inv.invoice_no = invoice_item.invoice_no
 AND inv.invoice_status >= 1
JOIN tom_trade_order_material material
  ON material.trade_order_no = seller_order.trade_order_no
 AND material.vehicle_material_code = seller_vo.vehicle_material_code
JOIN md_organization_customer_delivery_address addr
  ON addr.customer_no = seller_order.customer_no
 AND addr.delivery_short_name = seller_order.delivery_destination
JOIN lm_delivery_note dn
  ON dn.delivery_note_no = seller_vo.dn_no
 AND dn.data_status = 1
JOIN lm_delivery_note_info dni
  ON dni.delivery_note_no = dn.delivery_note_no
JOIN tom_vehicle_order oem_vo
  ON oem_vo.vin = seller_vo.vin
JOIN tom_trade_order oem_order
  ON oem_order.trade_order_no = oem_vo.trade_order_no
 AND oem_order.sales_company_no = '5000'
 AND oem_order.customer_no = '270165'
 AND oem_order.data_status = 4
WHERE seller_order.sales_company_no = 'L000'
  AND seller_order.shipper = 1
  AND seller_order.data_status = 4
  AND seller_vo.data_status = 8
  AND seller_vo.dn_time IS NOT NULL
  AND seller_vo.process_warehouse IS NOT NULL
  AND seller_vo.process_warehouse <> ''
  AND dni.gate_out_time IS NOT NULL
  AND dni.gate_in_received_time IS NOT NULL
  AND inv.invoice_date IS NOT NULL
  AND material.unit_price IS NOT NULL
ORDER BY seller_order.trade_order_no, seller_vo.vin;
```

`Invoice Price` 使用生产发票明细 `ic_invoice_item.price`，用于模板与生产对账；当前 `initInvoiceInfo` 仍不会把该列写入 JSON。

## 3、全量导出Logistics Sheet

结果列顺序与模板 `Logistics` Sheet 一致，一辆 VIN 一行。

```sql
SELECT DISTINCT
    seller_order.trade_order_no                         AS `Order No`,
    seller_vo.vin                                       AS `VIN`,
    DATE_FORMAT(seller_vo.dn_time, '%Y-%m-%d')          AS `DN Date`,
    DATE_FORMAT(dni.gate_out_time, '%Y-%m-%d')          AS `Gate Out Date`,
    DATE_FORMAT(dni.gate_in_received_time, '%Y-%m-%d')  AS `Received Date`,
    seller_vo.process_warehouse                         AS `From Warehouse`,
    seller_order.delivery_destination                   AS `Destination Short Name`,
    1                                                   AS `DN Type`
FROM tom_trade_order seller_order
JOIN tom_vehicle_order seller_vo
  ON seller_vo.trade_order_no = seller_order.trade_order_no
 AND seller_vo.vin IS NOT NULL
 AND seller_vo.vin <> ''
JOIN lm_delivery_note dn
  ON dn.delivery_note_no = seller_vo.dn_no
 AND dn.data_status = 1
JOIN lm_delivery_note_info dni
  ON dni.delivery_note_no = dn.delivery_note_no
JOIN tom_trade_order_material material
  ON material.trade_order_no = seller_order.trade_order_no
 AND material.vehicle_material_code = seller_vo.vehicle_material_code
JOIN md_organization_customer_delivery_address addr
  ON addr.customer_no = seller_order.customer_no
 AND addr.delivery_short_name = seller_order.delivery_destination
JOIN ic_invoice_item invoice_item
  ON invoice_item.vehicle_order_no = seller_vo.vehicle_order_no
JOIN ic_invoice inv
  ON inv.invoice_no = invoice_item.invoice_no
 AND inv.invoice_status >= 1
JOIN tom_vehicle_order oem_vo
  ON oem_vo.vin = seller_vo.vin
JOIN tom_trade_order oem_order
  ON oem_order.trade_order_no = oem_vo.trade_order_no
 AND oem_order.sales_company_no = '5000'
 AND oem_order.customer_no = '270165'
 AND oem_order.data_status = 4
WHERE seller_order.sales_company_no = 'L000'
  AND seller_order.shipper = 1
  AND seller_order.data_status = 4
  AND seller_vo.data_status = 8
  AND seller_vo.dn_time IS NOT NULL
  AND seller_vo.process_warehouse IS NOT NULL
  AND seller_vo.process_warehouse <> ''
  AND dni.gate_out_time IS NOT NULL
  AND dni.gate_in_received_time IS NOT NULL
  AND inv.invoice_date IS NOT NULL
  AND material.unit_price IS NOT NULL
ORDER BY seller_order.trade_order_no, seller_vo.vin;
```

`DN Type=1` 不是猜测的生产字段：当前导入服务创建销售 DN 时硬编码为订单 DN 类型 `1`，并不读取模板的 `DN Type` 值。三个日期仍使用生产真实时间；任一为空必须停止。

## 4、全量结果本地验收

生产查询结果导出后在本地完成以下检查，不再向运维提交新的 VIN 分批 SQL：

1. 从 0 号结果取得 `scope_check=PASS` 的 `seller_order_no` 清单。
2. `Order`、`Invoice`、`Logistics` 三份结果均按 `Order No` 只保留 PASS 订单。
3. 三个 Sheet 的 `(Order No, VIN)` 集合必须完全一致，且各自不得重复。
4. 再与业务期初库存 VIN 清单做交集/冲突检查；已作为当前库存入库且仍未销售的 VIN，不得生成历史 Seller 销售单和发票。
5. 任一订单缺地址、物料、价格、有效发票、DN、Gate Out 或 Received 日期，整单停止，不手工补假数据。
6. 校验通过后才分别粘贴到模板三个 Sheet，再调用 Excel 转 JSON 接口。

<details>
<summary>历史附录 B：旧批次三个 Sheet 数量验收（正常流程不执行）</summary>

## 历史批次三个Sheet数量与唯一性验收

每行必须为 `PASS`。

```sql
SELECT
    seller_order.trade_order_no AS order_no,
    COUNT(DISTINCT seller_vo.vin) AS order_vin_count,
    COUNT(DISTINCT CASE WHEN dn.data_status = 1 THEN seller_vo.vin END) AS logistics_vin_count,
    COUNT(DISTINCT CASE WHEN inv.invoice_status >= 1 THEN invoice_item.vin END) AS invoice_vin_count,
    COUNT(DISTINCT seller_vo.vehicle_material_code) AS material_count,
    CASE
      WHEN COUNT(DISTINCT seller_vo.vin)
             = COUNT(DISTINCT CASE WHEN dn.data_status = 1 THEN seller_vo.vin END)
       AND COUNT(DISTINCT seller_vo.vin)
             = COUNT(DISTINCT CASE WHEN inv.invoice_status >= 1 THEN invoice_item.vin END)
       AND COUNT(DISTINCT CASE
             WHEN seller_vo.dn_time IS NULL
               OR dni.gate_out_time IS NULL
               OR dni.gate_in_received_time IS NULL
             THEN seller_vo.vin END) = 0
      THEN 'PASS' ELSE 'STOP'
    END AS qty_check
FROM (
    SELECT DISTINCT oem_vo.vin
    FROM tom_vehicle_order oem_vo
    WHERE oem_vo.trade_order_no IN (<BATCH_OEM_ORDER_NO_LIST>)
      AND oem_vo.vin IS NOT NULL
) AS batch_vin
JOIN tom_vehicle_order seller_vo
  ON seller_vo.vin = batch_vin.vin
JOIN tom_trade_order seller_order
  ON seller_order.trade_order_no = seller_vo.trade_order_no
 AND seller_order.sales_company_no = 'L000'
 AND seller_order.shipper = 1
 AND seller_order.data_status = 4
LEFT JOIN lm_delivery_note dn
  ON dn.delivery_note_no = seller_vo.dn_no
 AND dn.data_status = 1
LEFT JOIN lm_delivery_note_info dni
  ON dni.delivery_note_no = dn.delivery_note_no
LEFT JOIN ic_invoice_item invoice_item
  ON invoice_item.vehicle_order_no = seller_vo.vehicle_order_no
LEFT JOIN ic_invoice inv
  ON inv.invoice_no = invoice_item.invoice_no
 AND inv.invoice_status >= 1
GROUP BY seller_order.trade_order_no
ORDER BY seller_order.trade_order_no;
```

Excel 粘贴后还要人工核对：

```text
Order 数据行数 = Logistics 数据行数 = Invoice 数据行数
三个 Sheet 的 Order No + VIN 集合完全一致
Order 的 Sales Company No 全部为 E000
Dealer SAP No、目的地、物料、价格、日期和发票号均未手改
模板中不存在任何南非参考行
```

</details>

## 5、UAT主数据映射检查

生产取数成功不代表 UAT 主数据已经具备。正式转 JSON 前，在 UAT 只读连接核对本批 Dealer 和目的地：

```sql
SELECT
    c.customer_no,
    c.customer_type,
    a.delivery_short_name,
    a.name AS delivery_name,
    a.country_code,
    a.address,
    COUNT(DISTINCT w.warehouse_no) AS mapped_warehouse_count,
    GROUP_CONCAT(DISTINCT w.warehouse_no ORDER BY w.warehouse_no SEPARATOR ',') AS mapped_warehouses
FROM md_organization_customer c
JOIN md_organization_customer_delivery_address a
  ON a.customer_no = c.customer_no
LEFT JOIN md_organization_customer_delivery_warehouse w
  ON w.customer_no = a.customer_no
 AND w.delivery_short_name = a.delivery_short_name
WHERE (c.customer_no, a.delivery_short_name) IN (
    <BATCH_DEALER_AND_DESTINATION_PAIR_LIST>
)
GROUP BY
    c.customer_no,
    c.customer_type,
    a.delivery_short_name,
    a.name,
    a.country_code,
    a.address
ORDER BY c.customer_no, a.delivery_short_name;
```

将参数替换为 1号 SQL 去重后的组合，例如：

```text
('DEALER001','DEST_A'),('DEALER002','DEST_B')
```

每个组合必须返回且 `country_code='MX'`。若 UAT Dealer 编号或目的地编码与生产不同，停止并形成明确映射表；不得直接改 Excel 猜测新编号。

## 6、未验证边界

这些 SQL 已按当前源码实体、Mapper 和初始化服务核对，但尚未在生产只读平台实际执行。全量首次运行必须从 0 号全量候选 SQL 开始；若出现字段不存在、超时、重复行或数量不一致，立即停止，保留报错和结果表头，再按真实生产库版本修订。不得把静态源码核对写成生产执行通过。
