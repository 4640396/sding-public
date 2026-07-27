---
title: "04 采购单保存 BigDecimal NPE"
type: log
status: reference
created: 2026-07-11
updated: 2026-07-11
sensitivity: internal
source:
project: DCS
---
# 采购单保存 BigDecimal NPE

## 现象

调用：

`POST /order/purchaseOrder/save`

堆栈：

`TradeOrderMaterialDao.java:183 → BigDecimal.add()`

## 根因

计算物料总金额时，以下字段仍为 `null`：

- `marketingAmount`
- `associationAmount`
- `csrAmount`
- `trainingAmount`

原代码先执行 `unitPrice.add(null)`，之后才读取 `MaterialPriceQuota`，计算顺序错误。

## 修改

文件：`TradeOrderMaterialDao.java`

1. 四项费用先初始化为 `BigDecimal.ZERO`；
2. 存在 `MaterialPriceQuota` 时用实际金额覆盖；
3. 最后计算物料总金额。

公式：

`materialAmount = (unitPrice + marketing + association + csr + training) × qty`

没有配额时：

`materialAmount = unitPrice × qty`

本次数据：

`10000 × 2 = 20000 MXN`

`BigDecimal.ZERO` 不改变有配额时的计算结果，只为无配额费用提供数学意义上的零值。



