---
title: "vehicle_instance stock_location 非空修复"
type: implementation
status: draft
created: 2026-07-17
updated: 2026-07-17
sensitivity: internal
project: DCS
repositories:
  - dcs-global-vehicle
sources:
  - dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/vehicle/domain/factory/VehicleFactory.java
  - dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/vehicle/infrastructure/dao/VehicleInstanceDao.java
  - dcs-global-vehicle/dcs-service/src/main/java/com/smil/globalvehicle/vehicle/infrastructure/repositoryimpl/VehicleRepositoryImpl.java
  - dcs-global-vehicle/document/04_SQL变更记录/feature-smmx-initProject-20260518.sql
---
# vehicle_instance stock_location 非空修复

## 现象

`VehicleInstanceDaoMapper.insertList` 批量插入 `vehicle_instance` 时失败：

```text
Column 'stock_location' cannot be null
```

生成的 `INSERT` 明确包含 `stock_location`，对应 Java 参数为 `null`。数据库字段虽然配置了 `NOT NULL DEFAULT 1`，但显式写入 `NULL` 不会触发默认值。

## 根因

`VehicleFactory.newVehicleFromFactory()` 创建车辆时未设置 `stockLocation`。`VehicleRepositoryImpl.saveAll()` 将领域对象转换为 `VehicleInstanceDao` 后通过 `insertList` 全字段插入，因此最终向非空字段传入了 `NULL`。

字段约定为：

- `1`：New Car
- `2`：Demo
- `3`：Fleet
- `4`：Reserved

## 实施内容

在 `VehicleFactory.newVehicleFromFactory()` 中显式设置新车默认库位：

```java
.stockLocation((byte) StockLocationEnum.NEW_CAR.getCode())
```

在 `VehicleInstanceDao(Vehicle vehicle)` 中增加持久化兜底：

```java
this.stockLocation = vehicle.getStockLocation() == null
        ? (byte) StockLocationEnum.NEW_CAR.getCode()
        : vehicle.getStockLocation();
```

该逻辑只把非法的 `null` 转为 `NEW_CAR(1)`；已经明确设置的 `2`、`3`、`4` 保持不变。数据库的 `DEFAULT 1` 继续作为省略字段时的数据库兜底，不能替代应用侧对全字段批量插入参数的初始化。

## 验证证据

- `git diff --check` 通过。
- 源码差异确认只涉及车辆工厂默认值和 DAO 持久化兜底。
- 执行 `mvn -o -pl dcs-service -DskipTests compile`，构建在源码编译前因本机缺少 Aspose、SAP JCo 和 `dcs-excel-bean` 私有依赖而失败。

## 未验证边界

- 尚未取得完整 Maven 编译通过证据。
- 尚未部署到目标环境验证实际 `insertList` 参数。
- 需要分别验证普通市场和 Mexico/SMMX 的车辆创建流程，确认默认 `New Car` 符合各市场业务口径。

在补齐私有依赖并完成集成验证前，本记录保持 `draft`，不得视为生产验收通过。
