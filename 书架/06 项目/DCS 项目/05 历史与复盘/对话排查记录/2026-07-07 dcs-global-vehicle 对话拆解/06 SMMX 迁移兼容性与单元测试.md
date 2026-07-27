---
title: "06 SMMX 迁移兼容性与单元测试"
type: log
status: reference
created: 2026-07-07
updated: 2026-07-11
sensitivity: internal
source:
project: DCS
---
# SMMX 迁移兼容性与单元测试

## Warehouse / Inventory byVin：global 与 smmx 差异

### SQL 层面

`smmx` 和 `global` 的 `queryAllByVin` 主 SQL 逻辑完全一致：

- SELECT 字段一致。
- FROM / JOIN 一致。
- WHERE `vein.business_status != 4` 一致。
- 动态条件一致：`vin1`、`vehicleMaterialCode1`、`customerName`、`customerCountry`、`warehouseNo`、`series`、`vehicleStatus`、`logisticsStatus`。
- 排序一致：`order by vein.create_time desc`。

唯一 SQL 文件层面的差异是 `resultType` 包名不同。

### 返回层面差异

global 不能算完全兼容 smmx，只能说主查询条件基本兼容。

#### global 多填充 warehouseName

global 在 `/pagelist/byVin` 中额外查仓库列表：

```java
warehouseService.findAllWarehouseNoBy()
```

并设置：

```java
pageVPO.setWarehouseName(warehouseMap.get(warehouseId));
```

smmx 只设置 `warehouseNo` 和 `groupName`。

#### Excel 导出列不同

smmx：

- 第 10 列是 `warehouseNo`
- 标题为 `Warehouse#`

 global：

- `warehouseNo` 被标成 `@ExlTransient`，不导出
- 多了 `warehouseName`
- 第 10 列标题变成 `Warehouse`

如果前端或调用方严格依赖导出列，二者不完全兼容。

#### localDescription 查询实现不同

smmx 在 Controller 内自己查 mapper：

- 若 `countryCode == null`，默认取当前用户组织国家。
- 使用当前用户组织 uuid + countryCode + materialCode 查询本地化描述。

 global 委托应用服务：

```java
organizationMaterialQueryApplicationService.findLocalDescriptionBy(countryCode, materialCode)
```

如果 global 服务内部规则与 smmx mapper 不一致，`localDescription` 可能不同。

## GateInVehicleQuery 新增 brand 字段

### 背景

接口新增：

```java
Optional<String> brand();
```

担心影响已有实现。

### 推荐写法

在接口里给默认实现：

```java
public interface GateInVehicleQuery extends CQRSQuery {

    default Optional<String> brand() {
        return Optional.empty();
    }

    Optional<String> dnNo();

    Optional<String> toWarehouse();

    Optional<Byte> gateInStatus();

    List<String> vins();
}
```

这样旧实现类不需要全部改，不会因为缺少 `brand()` 编译失败。

需要支持 brand 的实现类再覆盖：

```java
@Override
public Optional<String> brand() {
    return Optional.ofNullable(brand);
}
```

语义：老查询默认没有品牌过滤。

## DnGateQueryDto 构造方法访问权限

### 报错

```text
'DnGateQueryDto(...)' is not public in 'com.smil.globalvehicle.logistics.infrastructure.dto.DnGateQueryDto'. Cannot be accessed from outside package
```

### 原因

`DnGateQueryDto` 使用 Lombok `@Builder` 后，如果全参构造器不是 public，跨包直接访问构造方法会失败。

### 推荐修复

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class DnGateQueryDto {
    ...
}
```

必要时可加短注释：

```java
// Keep all-args constructor public for cross-package DTO projection access.
```

注释不能修复问题，关键是构造方法访问级别。

## 单元测试补充记录

### MxGatedInQueryResolverTest

位置：

```text
dcs-test/src/test/java/com/smil/globalvehicle/warehouse/userinterface/resolver/MxGatedInQueryResolverTest.java
```

覆盖：

- `GatedInQueryResolver.mxDnGateQuery(...)`
- VIN 空白值过滤
- `toWarehouseList` 多选仓库传递
- `deliveryWarehouseVendorCodeList` 传递
- `dnNo`、`gateInStatus`、`documentNo`、`dnType`、`alert`、`page` 映射
- page 为空默认 `1 / 10`

### MxWarehouseInventoryVehicleQueryControllerTest

位置：

```text
dcs-test/src/test/java/com/smil/globalvehicle/warehouse/userinterface/representation/MxWarehouseInventoryVehicleQueryControllerTest.java
```

覆盖：

- `mx/warehouse/inventoryVehicle/import/gateInByComfirm`
- `MxGateFileComfirmVPO` 转 `GateInDto`
- VIN、quality、gateInDate、customsReleaseNo、photo、photoName、notes 不丢失
- `inventoryVehicleApplicationService.mxGateInFileImport(...)`
- `mx/warehouse/inventoryVehicle/fileConfirmView` 文件读取失败时转换成 `WarehouseBusinessException`

### MxIcInvoiceControllerTest / AdditionalTest

位置：

```text
dcs-test/src/test/java/com/smil/globalvehicle/invoice/userinterface/restful/MxIcInvoiceControllerTest.java
dcs-test/src/test/java/com/smil/globalvehicle/invoice/userinterface/restful/MxIcInvoiceControllerAdditionalTest.java
```

覆盖方向：

- `sendInvoice` 空参数校验
- `cancelInvoices` 空列表返回空结果
- cancel 成功/失败聚合
- 重复错误按发票号合并展示
- 部分成功、部分失败时继续处理
- mapper 抛异常时继续处理下一条
- `ukCreateByVin` 成功/失败/锁释放
- 多条数据逐条加锁释放锁
- `vatRate`、`invoiceDate` 校验失败时不调用锁和开票服务

## 测试执行问题

曾尝试：

```bash
mvn -Dtest=MxIcInvoiceControllerTest test
```

但环境遇到内部依赖解析问题，例如：

- `smil-dcs-credit-api`
- `aspose-cells`
- `dcs-excel-bean`
- `sapjco3`
- `dcs-tpc-starter`

说明不是测试逻辑已跑失败，而是私有依赖 / 本地 Maven 环境不完整。







