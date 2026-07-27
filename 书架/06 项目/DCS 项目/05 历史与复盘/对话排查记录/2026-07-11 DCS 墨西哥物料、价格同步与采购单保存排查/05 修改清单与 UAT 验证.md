---
title: "05 修改清单与 UAT 验证"
type: log
status: reference
created: 2026-07-11
updated: 2026-07-11
sensitivity: internal
source:
project: DCS
---
# 修改清单与 UAT 验证

## 修改文件

- `vehicle/applicationservice/MxVehicleMaterialInfoMaintenanceApplicationService.java`
- `vehicle/userinterface/vdo/MxVehicleMaterialInfoMaintenanceCommandVDO.java`
- `tradeorder/infrastructure/dao/TradeOrderMaterialDao.java`

## 已完成检查

- `git diff --check` 已通过。
- 完整 Maven 编译受本地缺少内部依赖 `dcs-excel-bean:0.0.1` 和 SAP JCo 影响，未完成。

## UAT 待验证

- MX 物料新增；
- MX 物料修改；
- 品牌是否正确取自本地车系；
- `orderVehicleMaterial` 使用 `MXN` 查价；
- `purchaseOrder/save` 保存采购单；
- 订单物料总金额；
- 四项附加费用及价格配额快照。



