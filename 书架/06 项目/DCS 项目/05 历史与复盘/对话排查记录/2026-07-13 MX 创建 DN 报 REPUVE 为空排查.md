---
title: "MX 创建 DN 报 REPUVE 为空排查"
type: log
status: draft
created: 2026-07-13
updated: 2026-07-13
sensitivity: internal
source:
project: DCS
repositories:
  - dcs-global-vehicle
  - dcs-global-vehicle-client
  - dcs-smmx-vehicle
  - dcs-smmx-vehicle-client
---
# MX 创建 DN 报 REPUVE 为空排查

> [!warning] 结论状态
> REPUVE 校验逻辑和维护接口已通过当前代码确认。UAT 是否存在“旧 SMMX 页面写入与 global DN 创建读取不在同一服务或数据源”尚未通过运行时数据确认，因此本文标记为 `draft`。

## 现象

调用 MX DN Creation 接口：

```text
POST /api/dcs-global-api/dn/creation/mx/create
```

请求示例：

```json
{
  "capacity": "1",
  "transportationType": "1",
  "item": [
    {
      "vin": "<VIN>",
      "dnType": "1"
    }
  ]
}
```

接口报错：

```text
warehouse.repuve.is.blank.check
```

## 已确认原因

`dnType = 1` 代表 `Trade Order DN`。当前登录用户属于墨西哥销售公司时，创建 Trade Order DN 会校验车辆的 REPUVE 编号。

检查逻辑：

```java
if (vehicleOrderRepuveDto == null
        || StringUtils.isEmpty(vehicleOrderRepuveDto.getRepuve())) {
    throw new WarehouseBusinessException(
            "warehouse.repuve.is.blank.check",
            new String[]{vehicleOrderRepuveDto.getVin()});
}
```

底层通过整车订单和车辆实例关联读取 REPUVE：

```sql
SELECT vo.vehicle_order_no,
       vo.vin,
       vi.repuve
FROM tom_vehicle_order vo
JOIN vehicle_instance vi
  ON vo.vin = vi.vin
WHERE vo.vehicle_order_no IN (:vehicleOrderNos);
```

当 `vehicle_instance.repuve` 为 `NULL` 或空字符串时，抛出该错误。`capacity` 和 `transportationType` 不是本次错误的触发条件。

## dnType 的来源和含义

`/dn/creation/mx/create` 的 `dnType` 由调用方在每个 `item` 中直接传入，不是后端根据 VIN 自动计算。

```text
1 = Trade Order DN
2 = Transfer Order DN
```

Transfer Order DN 会跳过该 REPUVE 检查，但 DN 类型必须根据真实业务选择，不应为绕过校验而把 `dnType` 从 `1` 改为 `2`。

## REPUVE 维护页面

SMMX `Stock Detail-SMMX` 页面中的 **Update REPUVE Info** 是维护 REPUVE 和 REPUVE Date 的业务入口。

页面维护时需注意：

- 先确认报错中的 VIN，不要维护到相邻行的其他 VIN。
- 同时维护业务确认的 REPUVE# 和 REPUVE Date。
- 不应为通过校验而填入随意编号。
- 页面显示有值后，还需确认 DN 创建服务实际读取的 `vehicle_instance.repuve`。

## 迁移后接口差异

当前代码中存在两套 REPUVE 保存入口。

旧 SMMX 前端调用：

```text
POST /api/dcs-smmx-veh-api/warehouse/stock/detail/saveRepuveAndDate
```

迁移到 global 后的 MX 后端入口：

```text
POST /api/dcs-global-api/mx/warehouse/stock/detail/saveRepuveAndDate
```

两套实现都会通过车辆仓储更新 Vehicle 的 REPUVE 和 REPUVE Date。但 UAT 的网关路由、服务部署和数据源是否完全一致需要另行确认。

> [!important] 潜在风险
> 如果用旧 SMMX 页面维护，却调用 global 的 MX DN Creation 接口，可能出现页面已显示 REPUVE，但 global 服务读取的 `vehicle_instance.repuve` 仍为空的情况。这是当前待用 UAT 数据确认的推断，不是已验证根因。

## 排查 SQL

在 global 业务库检查目标 VIN：

```sql
SELECT vin,
       repuve,
       repuve_date
FROM vehicle_instance
WHERE vin = :vin;
```

同时检查 DN 使用的整车订单：

```sql
SELECT vo.trade_order_no,
       vo.vehicle_order_no,
       vo.vin,
       vi.repuve,
       vi.repuve_date
FROM tom_vehicle_order vo
JOIN vehicle_instance vi
  ON vi.vin = vo.vin
WHERE vo.vin = :vin;
```

判断方式：

1. global `vehicle_instance.repuve` 为空：先通过正确的 global MX 维护入口补齐业务数据。
2. global 库有值，页面也有值：继续核对 DN 实际找到的 `vehicle_order_no`、UAT 实际运行的代码版本及数据源。
3. 旧 SMMX 库有值而 global 库无值：可确认为迁移后页面与服务链路不一致。

## 验证步骤

1. 确认请求中的 VIN 和页面选中行一致。
2. 在 global 业务库查询 `vehicle_instance.repuve`。
3. 若为空，使用 global MX REPUVE 保存接口或对应的迁移后页面维护。
4. 保存后重新查询 `vehicle_instance`，不只根据页面显示判断。
5. 再次调用 `/dn/creation/mx/create`。
6. 如仍报错，核对 UAT 服务版本、网关路由和数据源。

## 关键代码

```text
dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/logistics/userinterface/restful/MxLmDeliveryNoteController.java
dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/logistics/application/service/DeliveryNoteApplicationService.java
dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/logistics/domain/valueobject/enums/DeliveryNoteTypeEnum.java
dcs-global-vehicle/dcs-service/src/main/resources/mappers/tom/VehicleOrderDaoMapper.xml
dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/warehouse/userinterface/restful/MxWarehouseStockDetailController.java
dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/warehouse/applicationservice/service/MxWarehouseStockDetailService.java
dcs-smmx-vehicle-client/src/pages/GLOBAL_VEHICLE/warehouse/stockDetail/stockDetail.vue
dcs-smmx-vehicle/dcs-service/src/main/java/com/smil/smmxvehicle/warehouse/userinterface/restful/WarehouseStockDetailController.java
```
