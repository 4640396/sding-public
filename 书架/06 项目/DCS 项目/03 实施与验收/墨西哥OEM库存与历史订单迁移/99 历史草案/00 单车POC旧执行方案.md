---
title: MX OEM STOCK 单车 POC 最终执行方案
type: runbook
status: deprecated
created: 2026-07-17
updated: 2026-07-18
sensitivity: internal
project: DCS
superseded_by: "[[../01 期初库存入库/03 批量导入执行方案]]"
---

# MX OEM STOCK 单车 POC 最终执行方案

> [!warning] 已被批量阶段方案替代
> 单车 POC 已完成采购发运导入和 Gate In 入库验收。本方案保留为单车历史过程，后续执行使用 [[../01 期初库存入库/03 批量导入执行方案]]；取数使用 [[../01 期初库存入库/02 批量生产只读取数]]。

## 准备文件

- SQL：[[01 单车生产只读SQL旧稿]] 当前仅保留为历史草案，不得复制执行；须先完成修订和生产只读续验。
- Excel：`C:\works\工作台\ZA采购发运期初数据1.xlsx`
- 当前只验证一辆车，物料固定为 `MCL2LC13K1CMBC`。
- 生产数据库只执行 SQL 文件中的 `SELECT`，不创建临时表，不执行写操作。

## 第一步：检查生产表结构

执行 SQL 第 0 段。

预期：返回以下表及关键字段：

- `tom_trade_order`
- `tom_vehicle_order`
- `lm_gds_delivery_interface`
- `lm_gds_delivery_interface_item`

若 SQL 报字段不存在或关键表没有返回，立即停止，把错误和结果发回来。

## 第二步：选择单车候选

执行 SQL 第 1 段。

优先选择同时满足以下条件的一行：

```text
order_vin_count = 1
invoice_vin_count = 1
latest_gds_missing_count = 0
```

记录该行的：

```text
source_order_no
vin
invoice_no
current_warehouse
```

不要修改生产数据。

## 第三步：替换 SQL 占位符

在 SQL 文件中执行全文替换：

```text
<POC_VIN>          → 第二步选中的 vin
<SOURCE_ORDER_NO>  → 第二步选中的 source_order_no
```

替换后搜索字符 `<`，与这两个占位符相关的结果必须为 0。

## 第四步：确认可以只导一辆

执行 SQL 第 2 段。

预期：只返回一行，并且：

```text
order_vin_count = 1
invoice_vin_count = 1
```

任一数量大于 1，停止，不制作单车模板，返回第二步重新选择候选。

## 第五步：导出订单 Sheet

执行 SQL 第 3 段。

预期：单车 POC 只返回一行，列顺序与无 VIN 订单模板 A:N 一致：

```text
客户订单号
销售定价日期
下单日期
DEALER SAP NO
DELIVERY SHORT NAME
Material Code
开票日期
INVOICE NO
Incoterm
Price
Currency
Material Description
Interior Description
Exterior Description
```

操作：

1. 复制 `ZA采购发运期初数据1.xlsx` 作为新的 POC 文件，不要直接修改原文件。
2. 清除“订单”Sheet 中原有示例数据，只保留标题。
3. 将第 3 段唯一结果行粘贴到第 2 行。
4. 不复制 SQL 查询工具生成的行号或列标题。
5. 不增加 VIN 列，不调整列顺序。

## 第六步：检查发运字段

执行 SQL 第 5 段。

预期：只返回一行，`missing_fields` 为空。

如果 `missing_fields` 包含任何字段，立即停止，不使用当前日期、发票日期或其他日期代替，把查询结果发回来。

## 第七步：导出发运 Sheet

只有第六步通过后，执行 SQL 第 4 段。

预期：只返回一行，包含同一订单、同一物料和同一 VIN。

操作：

1. 清除 POC 文件“发运”Sheet 中原有示例数据，只保留标题。
2. 将第 4 段唯一结果行粘贴到第 2 行。
3. 不复制 SQL 行号或列标题。
4. 不修改 VIN、订单号、物料、发票号和业务日期。

## 第八步：数量一致性检查

执行 SQL 第 5A 段。

单车 POC 预期：

```text
order_sheet_row_count = 1
shipping_vin_count = 1
qty_check = PASS
```

不是 `PASS` 就停止，不调用接口。

然后人工核对 Excel：

```text
订单 Sheet 客户订单号 = 发运 Sheet 销售订单号
订单 Sheet Material Code = 发运 Sheet 主物料号
订单 Sheet 1 行 = 发运 Sheet 1 个去重 VIN
```

## 第九步：Excel 转 JSON

调用：

```text
POST /ops/smza-order/initGdsShippingData
file=<刚制作的 POC Excel>
path=<UAT应用服务器可写目录，末尾保留路径分隔符>
salesCompanyNo=<采购订单卖方销售公司编号；当前接口要求总部 5000>
```

接口会在服务器 `path` 下生成：

```text
PurchaseAndShipping.json
```

先打开 JSON 检查，不要立即调用 save：

```text
订单数组数量 = 1
items 数量 = 1
items[0].qty = 1
deliveryItemVos 数量 = 1
订单号、物料、VIN、发票号与 Excel 一致
```

任一项不一致就停止，把脱敏后的 JSON 结构和问题发回来。

## 第十步：检查 UAT 不存在重复数据

在 UAT 使用订单号和 VIN 查询：

```text
同订单号不存在
同 VIN 不存在
同 VIN 库存记录不存在
```

存在任一记录就停止，不调用 save。

## 第十一步：导入采购订单和发运

调用一次：

```text
POST /ops/smpOrder/save
Content-Type: application/json
Body=<PurchaseAndShipping.json 完整 JSON 数组>
```

保存请求时间、HTTP 状态、响应正文和日志关键字。

接口超时或结果不明确时禁止立即重试，先按订单号和 VIN 查询是否已经部分落库。

## 第十二步：Gate In 前检查

确认 UAT 同时存在：

```text
采购订单及一个物料，qty = 1
该 VIN 的车辆订单/车辆记录
该 VIN 对应的 shipping、ASN 或 DN
目的仓与第二步 current_warehouse 对应
该 VIN 的待入库记录
```

任一项缺失或目的仓错误，不执行 Gate In。

## 第十三步：准备并执行 Gate In

执行 SQL 第 6 段取得 Gate In 参考数据，并与业务库存清单中的原 Gate In 日期核对。

Gate In 模板只保留一辆车：

```text
VIN = 本次 POC VIN
Quality = 1
Gate In Date = 经业务清单确认的原日期
Customs Release = 有可靠来源才填写，否则留空
Notes = MX OEM STOCK POC
```

然后在 UAT 页面执行批量 Gate In。

## 第十四步：最终验收

按 VIN 验证：

```text
Gate In 成功且无失败明细
VIN 唯一
物料为 MCL2LC13K1CMBC
最终仓库正确
物流状态、业务状态、质量状态符合 Gate In 后规则
采购订单、发运/DN、库存链路可以追溯
```

全部通过后，才讨论扩大到其余车辆。

## 任何阶段立即停止的条件

- SQL 报表或字段不存在。
- 订单或发票包含多个 VIN。
- `missing_fields` 不为空。
- `qty_check` 不是 `PASS`。
- JSON 中订单数量、`qty` 或发运 VIN 数不等于 1。
- UAT 已存在同订单号或 VIN。
- save 超时、部分成功或响应不明确。
- DN/待入库记录缺失或目的仓错误。

本页现为 `deprecated`，仅保留单车 POC 历史过程；单车已经在 UAT 完成采购发运导入和 Gate In，后续以 [[../01 期初库存入库/03 批量导入执行方案]] 为准。
