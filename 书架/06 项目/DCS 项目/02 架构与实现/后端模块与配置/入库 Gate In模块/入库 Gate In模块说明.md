---
title: "入库 Gate In模块说明"
type: project-knowledge
status: verified
created: 2026-07-11
updated: 2026-07-11
sensitivity: internal
source:
project: DCS
---
# 入库 Gate In模块说明

## 模块范围

这里沉淀 DCS 中和仓库入库 Gate In 相关的后端排查知识，主要包括：

- VIN 入库 Gate In。
- 批量导入 Gate In。
- Teleport Gate In。
- 入库仓库、保税属性、CDN 确认状态等业务校验。
- Gate In 相关业务异常的排查 SQL 和代码入口。

## 常见问题

- [[保税车与非保税车入库仓库类型校验|保税车与非保税车入库仓库类型校验]]

## 关键代码入口

```text
dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/warehouse/applicationservice/service/InventoryVehicleApplicationService.java
dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/logisticsv2/applicationservice/biz/TeleportGateInNoteApplicationService.java
dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/vehicle/domain/valueobject/Bonded.java
dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/masterdata/domain/valueobject/WarehouseBonded.java
```

## 关键概念

- `Bonded.BONDED`：车辆是保税/未清关状态。
- `Bonded.FREE`：车辆是非保税/已清关/free 状态。
- `WarehouseBonded.BONDED`：保税仓。
- `WarehouseBonded.NONBONDED`：非保税仓，`FREE` 和 `NON-BONDED` 都会被解析成这个值。
- Gate In：车辆入库。






