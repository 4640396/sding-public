---
title: MX OEM STOCK 生产只读取数
type: sql-runbook
status: deprecated
created: 2026-07-17
updated: 2026-07-18
sensitivity: internal
project: DCS
superseded_by: "[[../01 期初库存入库/02 批量生产只读取数]]"
---

# MX OEM STOCK 生产只读取数

> [!warning] 已被批量 SQL 替代
> 本 SQL 是单车 POC 历史草案。单车成功证明 `sales_company_no=5000` 是总部卖方、生产 `customer_no=270165` 是墨西哥买方，两者并非同一字段的环境编码冲突；真正需要修订的是单车边界、`g.etd` 漏检和生产到 UAT 的客户字段映射。后续使用 [[../01 期初库存入库/02 批量生产只读取数]]。

下面的 SQL 仅保留为历史草案。方案尚未在生产只读平台完整执行，当前不得复制到数据库客户端运行。

```sql
/*
  MX OEM STOCK 单车 POC（生产只读）
  MySQL 5.7 兼容：无 WITH、无窗口函数、无临时表、无写操作。

  固定 POC 物料：MCL2LC13K1CMBC
  执行第 1 段选中候选后，把以下占位符替换为查询结果：
    <POC_VIN>
    <SOURCE_ORDER_NO>
*/

/* 0. 表和关键字段预检；四张表都应返回。 */
SELECT table_name, column_name
FROM information_schema.columns
WHERE table_schema = DATABASE()
  AND (
       (table_name = 'tom_trade_order' AND column_name IN
        ('trade_order_no','pricing_date','customer_no','delivery_destination',
         'incoterm_code','currency','sales_company_no','source_code','data_status'))
    OR (table_name = 'tom_vehicle_order' AND column_name IN
        ('trade_order_no','vin','vehicle_material_code'))
    OR (table_name = 'lm_gds_delivery_interface' AND column_name IN
        ('id','im_time','invoice_no','invoice_date','send_time','submit_date','bl_date'))
    OR (table_name = 'lm_gds_delivery_interface_item' AND column_name IN
        ('lm_delivery_interface_id','vehicle_vin','material_no','engine_code'))
  )
ORDER BY table_name, column_name;


/*
  1. 选择单车 POC 候选。

  选择规则：
  - data_status=4 是 Approved，来自 tom_trade_order 表注释，不是墨西哥/全球枚举差异。
  - order_vin_count=1、invoice_vin_count=1。
  - latest_gds_missing_count=0 最理想；大于 0 先看第 5 段，不要编造日期。
  - current_warehouse 是当前库存仓；订单模板目的地仍取订单的 delivery_destination。
*/
SELECT
    tor.trade_order_no AS source_order_no,
    vo.vin,
    vo.vehicle_material_code AS material_code,
    g.invoice_no,
    tor.delivery_destination,
    wiv.warehouse_no AS current_warehouse,
    vi.logistics_status,
    vi.business_status,
    vi.quality_status,
    tor.sales_company_no,
    tor.customer_no,
    tor.source_code,
    tor.data_status,
    (SELECT COUNT(DISTINCT vo2.vin)
       FROM tom_vehicle_order vo2
      WHERE vo2.trade_order_no = tor.trade_order_no) AS order_vin_count,
    (SELECT COUNT(DISTINCT gi3.vehicle_vin)
       FROM lm_gds_delivery_interface_item gi3
       JOIN lm_gds_delivery_interface g3
         ON g3.id = gi3.lm_delivery_interface_id
      WHERE g3.invoice_no = g.invoice_no) AS invoice_vin_count,
    ((g.id IS NULL) +
     (g.invoice_date IS NULL) +
     (g.send_time IS NULL) +
     (g.submit_date IS NULL) +
     (g.bl_date IS NULL) +
     (gi.offline_date IS NULL) +
     (gi.call_car_date IS NULL)) AS latest_gds_missing_count
FROM tom_vehicle_order vo
JOIN tom_trade_order tor
  ON tor.trade_order_no = vo.trade_order_no
JOIN vehicle_instance vi
  ON vi.vin = vo.vin
LEFT JOIN warehouse_inventory_vehicle wiv
  ON wiv.vin = vo.vin
 AND wiv.status IN (1, 2)
LEFT JOIN lm_gds_delivery_interface g
  ON g.id = (
     SELECT g2.id
     FROM lm_gds_delivery_interface_item gi2
     JOIN lm_gds_delivery_interface g2
       ON g2.id = gi2.lm_delivery_interface_id
     WHERE gi2.vehicle_vin = vo.vin
     ORDER BY g2.im_time DESC, g2.id DESC
     LIMIT 1
 )
LEFT JOIN lm_gds_delivery_interface_item gi
  ON gi.lm_delivery_interface_id = g.id
 AND gi.vehicle_vin = vo.vin
WHERE vo.vehicle_material_code = 'MCL2LC13K1CMBC'
  AND tor.sales_company_no = '5000'
  AND tor.customer_no = '270165'
  AND tor.source_code = 1
  AND tor.data_status = 4
  AND vi.logistics_status = 2
  AND vi.quality_status = 1
ORDER BY
    order_vin_count,
    invoice_vin_count,
    latest_gds_missing_count,
    tor.trade_order_no,
    vo.vin
LIMIT 100;


/* 2. 对选中的 VIN 做最终边界核验；应只返回一行，两个数量都应为 1。 */
SELECT
    tor.trade_order_no AS source_order_no,
    vo.vin,
    vo.vehicle_material_code AS material_code,
    g.invoice_no,
    wiv.warehouse_no AS current_warehouse,
    (SELECT COUNT(DISTINCT vo2.vin)
       FROM tom_vehicle_order vo2
      WHERE vo2.trade_order_no = tor.trade_order_no) AS order_vin_count,
    (SELECT COUNT(DISTINCT gi3.vehicle_vin)
       FROM lm_gds_delivery_interface_item gi3
       JOIN lm_gds_delivery_interface g3
         ON g3.id = gi3.lm_delivery_interface_id
      WHERE g3.invoice_no = g.invoice_no) AS invoice_vin_count
FROM tom_vehicle_order vo
JOIN tom_trade_order tor
  ON tor.trade_order_no = vo.trade_order_no
LEFT JOIN warehouse_inventory_vehicle wiv
  ON wiv.vin = vo.vin
 AND wiv.status IN (1, 2)
LEFT JOIN lm_gds_delivery_interface g
  ON g.id = (
     SELECT g2.id
     FROM lm_gds_delivery_interface_item gi2
     JOIN lm_gds_delivery_interface g2
       ON g2.id = gi2.lm_delivery_interface_id
     WHERE gi2.vehicle_vin = vo.vin
     ORDER BY g2.im_time DESC, g2.id DESC
     LIMIT 1
 )
LEFT JOIN lm_gds_delivery_interface_item gi
  ON gi.lm_delivery_interface_id = g.id
 AND gi.vehicle_vin = vo.vin
WHERE vo.vin = '<POC_VIN>'
  AND tor.trade_order_no = '<SOURCE_ORDER_NO>';


/*
  3. 导出业务确认的无 VIN“订单”Sheet（A:N）。
  VIN 不输出，但查询仍保留 tom_vehicle_order 的逐车行粒度；
  不得增加 DISTINCT 或 GROUP BY。接口按“订单号+物料”的行数计算 qty。
  单车 POC 应返回一行。复制数据行，不覆盖模板标题。
*/
SELECT
    tor.trade_order_no                         AS `客户订单号`,
    DATE_FORMAT(tor.pricing_date, '%Y-%m-%d') AS `销售定价日期`,
    DATE_FORMAT(tor.pricing_date, '%Y-%m-%d') AS `下单日期`,
    tor.customer_no                            AS `DEALER SAP NO`,
    tor.delivery_destination                   AS `DELIVERY SHORT NAME`,
    tm.vehicle_material_code                   AS `Material Code`,
    DATE_FORMAT(g.invoice_date, '%Y-%m-%d')    AS `开票日期`,
    g.invoice_no                               AS `INVOICE NO`,
    tor.incoterm_code                          AS `Incoterm`,
    tm.unit_price                              AS `Price`,
    tor.currency                               AS `Currency`,
    tm.material_description                    AS `Material Description`,
    tm.interior_description                    AS `Interior Description`,
    tm.exterior_description                    AS `Exterior Description`
FROM tom_vehicle_order vo
JOIN tom_trade_order tor
  ON tor.trade_order_no = vo.trade_order_no
JOIN tom_trade_order_material tm
  ON tm.trade_order_no = tor.trade_order_no
 AND tm.vehicle_material_code = vo.vehicle_material_code
LEFT JOIN lm_gds_delivery_interface g
  ON g.id = (
     SELECT g2.id
     FROM lm_gds_delivery_interface_item gi2
     JOIN lm_gds_delivery_interface g2
       ON g2.id = gi2.lm_delivery_interface_id
     WHERE gi2.vehicle_vin = vo.vin
     ORDER BY g2.im_time DESC, g2.id DESC
     LIMIT 1
 )
LEFT JOIN lm_gds_delivery_interface_item gi
  ON gi.lm_delivery_interface_id = g.id
 AND gi.vehicle_vin = vo.vin
WHERE vo.vin = '<POC_VIN>'
  AND tor.trade_order_no = '<SOURCE_ORDER_NO>';


/*
  4. 导出原始 ZA采购发运期初数据.xlsx 的“发运”Sheet。
  一辆车一行。单车 POC 应返回一行。
  日期保持真实 NULL；不要在生产查询中使用 COALESCE 伪造业务日期。
*/
SELECT DISTINCT
    tor.trade_order_no                         AS `销售订单号`,
    vo.vin                                     AS `Vehicle VIN`,
    gi.engine_code                             AS `发动机码`,
    gi.material_no                             AS `主物料号`,
    DATE_FORMAT(gi.offline_date, '%Y-%m-%d')   AS `下线日期`,
    DATE_FORMAT(gi.call_car_date, '%Y-%m-%d')  AS `CALL车日期`,
    gi.delivery_no                             AS `发运单编号`,
    g.invoice_no                               AS `议付发票号`,
    gi.sap_out_delivery_no                     AS `外向交货单号`,
    g.delivery_plan                            AS `发运计划`,
    g.delivery_play_remark                     AS `发运计划备注`,
    DATE_FORMAT(g.send_time, '%Y-%m-%d')       AS `TT寄单日期`,
    'Others'                                   AS `区域`,
    g.country                                  AS `国家`,
    CONCAT_WS(' ', g.vessel_name, g.voyage)   AS `船名航次`,
    DATE_FORMAT(g.etd, '%Y-%m-%d')             AS `预计起运日期`,
    g.port_of_departure                        AS `起运港`,
    g.port_of_destination                      AS `目的港`,
    DATE_FORMAT(g.submit_date, '%Y-%m-%d')     AS `收提单日期`,
    DATE_FORMAT(g.bl_date, '%Y-%m-%d')         AS `提单日期`,
    g.bl_no                                    AS `提单号`,
    g.customer_no                              AS `客户编号`,
    g.ship_to_party_no                         AS `送达方编号`,
    g.invoice_amount                           AS `发票金额`,
    DATE_FORMAT(g.invoice_date, '%Y-%m-%d')    AS `开票日期`,
    gi.declaration_of_ch_name                  AS `报关中文名称`,
    gi.declaration_of_en_name                  AS `报关英文名称`
FROM tom_vehicle_order vo
JOIN tom_trade_order tor
  ON tor.trade_order_no = vo.trade_order_no
JOIN lm_gds_delivery_interface_item gi
  ON gi.vehicle_vin = vo.vin
JOIN lm_gds_delivery_interface g
  ON g.id = gi.lm_delivery_interface_id
 AND g.id = (
     SELECT g2.id
     FROM lm_gds_delivery_interface_item gi2
     JOIN lm_gds_delivery_interface g2
       ON g2.id = gi2.lm_delivery_interface_id
     WHERE gi2.vehicle_vin = vo.vin
     ORDER BY g2.im_time DESC, g2.id DESC
     LIMIT 1
 )
WHERE vo.vin = '<POC_VIN>'
  AND tor.trade_order_no = '<SOURCE_ORDER_NO>';


/*
  5. 发运必填/转换日期检查。
  missing_fields 为空才能直接执行 initGdsShippingData。
  当前转换代码会直接解析 7 个日期字符串，任一个 NULL 都会失败。
*/
SELECT
    vo.vin,
    g.id AS gds_id,
    g.invoice_no,
    g.im_time AS gds_receive_time,
    CONCAT_WS(', ',
      IF(gi.offline_date IS NULL, 'offlineDate', NULL),
      IF(gi.call_car_date IS NULL, 'callCarDate', NULL),
      IF(g.send_time IS NULL, 'ttSendDocumentDate', NULL),
      IF(g.etd IS NULL, 'startTransportDate', NULL),
      IF(g.submit_date IS NULL, 'dateOfBillOfLoading', NULL),
      IF(g.bl_date IS NULL, 'dateOfBill', NULL),
      IF(g.invoice_date IS NULL, 'invoiceDate', NULL)
    ) AS missing_fields
FROM tom_vehicle_order vo
JOIN lm_gds_delivery_interface_item gi
  ON gi.vehicle_vin = vo.vin
JOIN lm_gds_delivery_interface g
  ON g.id = gi.lm_delivery_interface_id
 AND g.id = (
     SELECT g2.id
     FROM lm_gds_delivery_interface_item gi2
     JOIN lm_gds_delivery_interface g2
       ON g2.id = gi2.lm_delivery_interface_id
     WHERE gi2.vehicle_vin = vo.vin
     ORDER BY g2.im_time DESC, g2.id DESC
     LIMIT 1
 )
WHERE vo.vin = '<POC_VIN>';


/*
  5A. 模板数量一致性核验。
  单车 POC 两个数量都必须为 1。
  批量时应按“订单号+物料”核对：订单 Sheet 行数 = 发运 Sheet 去重 VIN 数。
*/
SELECT
    tor.trade_order_no AS order_no,
    vo.vehicle_material_code AS material_code,
    COUNT(*) AS order_sheet_row_count,
    COUNT(DISTINCT gi.vehicle_vin) AS shipping_vin_count,
    CASE WHEN COUNT(*) = COUNT(DISTINCT gi.vehicle_vin)
         THEN 'PASS' ELSE 'FAIL' END AS qty_check
FROM tom_vehicle_order vo
JOIN tom_trade_order tor
  ON tor.trade_order_no = vo.trade_order_no
LEFT JOIN lm_gds_delivery_interface g
  ON g.id = (
     SELECT g2.id
     FROM lm_gds_delivery_interface_item gi2
     JOIN lm_gds_delivery_interface g2
       ON g2.id = gi2.lm_delivery_interface_id
     WHERE gi2.vehicle_vin = vo.vin
     ORDER BY g2.im_time DESC, g2.id DESC
     LIMIT 1
  )
LEFT JOIN lm_gds_delivery_interface_item gi
  ON gi.lm_delivery_interface_id = g.id
 AND gi.vehicle_vin = vo.vin
WHERE tor.trade_order_no = '<SOURCE_ORDER_NO>'
  AND vo.vin = '<POC_VIN>'
GROUP BY tor.trade_order_no, vo.vehicle_material_code;


/* 6. Gate In 模板数据：只在采购订单/save 和待入库验证通过后使用。 */
SELECT
    vi.vin AS `VIN`,
    CASE WHEN vi.quality_status = 2 THEN '2' ELSE '1' END AS `Quality`,
    DATE_FORMAT(wivg.gate_in_date, '%Y-%m-%d') AS `Gate In Date`,
    '' AS `Customs Release #（optional）`,
    '' AS `photo`,
    'MX OEM STOCK POC' AS `Notes`
FROM vehicle_instance vi
LEFT JOIN warehouse_inventory_vehicle wiv
  ON wiv.vin = vi.vin
 AND wiv.status IN (1, 2)
LEFT JOIN warehouse_inventory_vehicle_gated_in_note wivg
  ON wivg.inventory_vehicle_no = wiv.inventory_vehicle_no
WHERE vi.vin = '<POC_VIN>'
ORDER BY wivg.gate_in_date DESC, wivg.create_time DESC
LIMIT 1;

```
