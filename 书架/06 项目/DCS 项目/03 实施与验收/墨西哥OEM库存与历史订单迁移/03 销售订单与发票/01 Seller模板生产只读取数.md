---
title: MX Seller运维全量导出SQL
type: sql-runbook
status: accepted
created: 2026-07-18
updated: 2026-07-25
sensitivity: internal
project: DCS
sources:
  - $env:WORKS_ROOT/initdata/Order/经销商采购导入模版.xlsx
  - "[[02 Seller批量执行方案]]"
  - "[[../99 历史草案/Seller生产取数与SQL排查记录]]"
---

# MX Seller运维全量导出SQL

> [!important] 运维只执行本页三条 SQL
> 在生产只读库 `smil_dcs_smmx_vehicle` 分别执行 1、2、3 号 SQL，完整导出为 `01_Order.xlsx`、`02_Invoice.xlsx`、`03_Logistics.xlsx`。不得增加时间条件、修改组织与状态条件或只导出页面最大显示行数。0号验证、错误版本和旧批次排查已移入历史草案，不在本页执行。

三条 SQL 使用完全相同的候选条件：生产 `L000` Seller 已审批订单、车辆状态 `8=Invoiced`、有效 DN、Gate Out、Received、有效发票、物料/价格/地址完整，并且每个VIN唯一追溯到一个`5000 → 270165`的已审批OEM订单。

## 1、导出 Order Sheet

```sql
SELECT DISTINCT
    seller_order.trade_order_no                        AS `Order No`,
    'E000'                                             AS `Sales Company No`,
    seller_order.incoterm_code                         AS `Incoterm Code`,
    DATE_FORMAT(seller_order.create_time, '%Y-%m-%d')  AS `Order Date`,
    seller_order.delivery_destination                  AS `DELIVERY SHORT NAME`,
    seller_order.customer_no                           AS `DEALER SAP NO`,
    seller_order.delivery_destination                  AS `DELIVERY SHORT NAME`,
    addr.name                                          AS `Delivery Name`,
    addr.country_code                                  AS `Country Code`,
    addr.address                                       AS `ADDRESS`,
    seller_vo.vin                                      AS `VIN`,
    seller_order.order_type                            AS `Order Type`,
    material.unit_price                                AS `Sales Price`,
    seller_order.currency                              AS `Currency`,
    seller_vo.vehicle_material_code                    AS `Material Code`,
    material.material_description                      AS `Material Description`,
    material.interior_description                      AS `Interior Description`,
    material.exterior_description                      AS `Exterior Description`
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
  AND 1 = (
      SELECT COUNT(DISTINCT oem_order_check.trade_order_no)
      FROM tom_vehicle_order oem_vo_check
      JOIN tom_trade_order oem_order_check
        ON oem_order_check.trade_order_no = oem_vo_check.trade_order_no
       AND oem_order_check.sales_company_no = '5000'
       AND oem_order_check.customer_no = '270165'
       AND oem_order_check.data_status = 4
      WHERE oem_vo_check.vin = seller_vo.vin
  )
  AND seller_order.create_time IS NOT NULL
  AND seller_vo.process_warehouse IS NOT NULL
  AND seller_vo.process_warehouse <> ''
  AND dni.gate_out_time IS NOT NULL
  AND dni.gate_in_received_time IS NOT NULL
  AND inv.invoice_date IS NOT NULL
  AND material.unit_price IS NOT NULL
ORDER BY seller_order.trade_order_no, seller_vo.vin;
```

结果列顺序与模板 `Order` Sheet 一致。模板中的两个 `DELIVERY SHORT NAME` 均输出相同的生产真实值；`E000` 是目标环境组织编码映射，生产源筛选仍为 `L000`。

## 2、导出 Invoice Sheet

```sql
SELECT DISTINCT
    seller_order.trade_order_no                              AS `Order No`,
    seller_vo.vin                                            AS `VIN`,
    COALESCE(NULLIF(inv.biz_invoice_no, ''), inv.invoice_no) AS `Invoice No`,
    DATE_FORMAT(inv.invoice_date, '%Y-%m-%d')                AS `Invoice Date`,
    invoice_item.price                                       AS `Invoice Price`
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
  AND 1 = (
      SELECT COUNT(DISTINCT oem_order_check.trade_order_no)
      FROM tom_vehicle_order oem_vo_check
      JOIN tom_trade_order oem_order_check
        ON oem_order_check.trade_order_no = oem_vo_check.trade_order_no
       AND oem_order_check.sales_company_no = '5000'
       AND oem_order_check.customer_no = '270165'
       AND oem_order_check.data_status = 4
      WHERE oem_vo_check.vin = seller_vo.vin
  )
  AND seller_order.create_time IS NOT NULL
  AND seller_vo.process_warehouse IS NOT NULL
  AND seller_vo.process_warehouse <> ''
  AND dni.gate_out_time IS NOT NULL
  AND dni.gate_in_received_time IS NOT NULL
  AND inv.invoice_date IS NOT NULL
  AND material.unit_price IS NOT NULL
ORDER BY seller_order.trade_order_no, seller_vo.vin;
```

`Invoice Price` 用于模板与生产对账；当前 `initInvoiceInfo` 不会把该列写入 JSON。

## 3、导出 Logistics Sheet

```sql
SELECT DISTINCT
    seller_order.trade_order_no                        AS `Order No`,
    seller_vo.vin                                      AS `VIN`,
    DATE_FORMAT(dni.gate_out_time, '%Y-%m-%d')         AS `DN Date`,
    DATE_FORMAT(dni.gate_out_time, '%Y-%m-%d')         AS `Gate Out Date`,
    DATE_FORMAT(dni.gate_in_received_time, '%Y-%m-%d') AS `Received Date`,
    seller_vo.process_warehouse                        AS `From Warehouse`,
    seller_order.delivery_destination                  AS `Destination Short Name`,
    1                                                  AS `DN Type`
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
  AND 1 = (
      SELECT COUNT(DISTINCT oem_order_check.trade_order_no)
      FROM tom_vehicle_order oem_vo_check
      JOIN tom_trade_order oem_order_check
        ON oem_order_check.trade_order_no = oem_vo_check.trade_order_no
       AND oem_order_check.sales_company_no = '5000'
       AND oem_order_check.customer_no = '270165'
       AND oem_order_check.data_status = 4
      WHERE oem_vo_check.vin = seller_vo.vin
  )
  AND seller_order.create_time IS NOT NULL
  AND seller_vo.process_warehouse IS NOT NULL
  AND seller_vo.process_warehouse <> ''
  AND dni.gate_out_time IS NOT NULL
  AND dni.gate_in_received_time IS NOT NULL
  AND inv.invoice_date IS NOT NULL
  AND material.unit_price IS NOT NULL
ORDER BY seller_order.trade_order_no, seller_vo.vin;
```

`DN Type=1` 与当前导入服务创建销售 DN 时的固定订单 DN 类型一致。生产实测中 `lm_delivery_note.create_time` 存在晚于 Gate Out/Received 的后补记录，不能作为历史 DN 业务日期；当前导入服务只用 `DN Date` 非空判断是否创建 DN，并不会把该值写成 DN 历史时间，因此本 SQL 使用可靠的 `gate_out_time` 同时填充 `DN Date` 和 `Gate Out Date`。真实 Gate Out 与 Received 日期仍分别写入目标业务事件。

## 收到三个 Excel 后验收

```text
每个文件的 (Order No, VIN) 不重复
三个文件的 (Order No, VIN) 集合完全一致
三个文件均为完整导出，不受页面最大显示行数截断
```

通过后，分别粘贴到 `经销商采购导入模版.xlsx` 的 `Order`、`Invoice`、`Logistics` Sheet，再按 [[02 Seller批量执行方案]] 拆批导入 UAT。
