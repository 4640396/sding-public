---
title: MX Gate Out 图片上传、保存与查询迁移差异
type: troubleshooting
status: verified
created: 2026-07-15
updated: 2026-07-15
sensitivity: internal
project: DCS
module: warehouse-gate-out
date: 2026-07-15
tags:
  - DCS
  - Mexico
  - GateOut
  - AzureBlob
  - migration
---

# MX Gate Out 图片上传、保存与查询迁移差异

## 最终结论

旧 SMMX Gate Out 图片能力迁入全球整车后，存在三类相互独立的迁移差异：

1. **上传返回值不同**：旧 SMMX 返回相对路径，全球整车旧上传方法返回绝对 URL。
2. **列表查询映射遗漏**：全球共享查询误用 `resultType`，无法组装 `inventoryVehicleStatus` 和嵌套的 `gateOutDto`。
3. **Gate Out Confirm 保存遗漏**：全球 `GateOutNoteFactory` 构建领域对象时漏传 `photo`、`photoName`。

三者会分别造成：上传后前端误判新旧图片、列表关联不到库存记录、Gate Out 成功但数据库图片字段仍为空。

## 涉及接口

- 旧 SMMX 上传：`/dcs-smmx-veh-api/warehouse/inventoryVehicle/uploadPhoto`
- 全球整车 MX 上传：`/dcs-global-api/mx/warehouse/inventoryVehicle/uploadPhoto`
- MX Gate Out Confirm：`/dcs-global-api/mx/warehouse/inventoryVehicle/import/gateOutByComfirm`
- MX 单独保存照片：`/dcs-global-api/mx/warehouse/inventoryVehicle/savePhoto`
- MX Gate Out 列表：`/dcs-global-api/mx/warehouse/inventoryVehicle/pagelist/gateOut`

## 字段含义

- `photo`：数据库保存的原始图片路径，约定为 `container/blob` 相对路径。
- `photoAsbUrl`：查询时根据 `photo` 生成的可访问绝对地址；`Asb` 是历史拼写，实际表示 `Abs/Absolute URL`。
- `photoName`：上传文件原名，可空，不参与 Gate Out 核心业务判断。
- `photoCopy`：旧 MX 前端用于备份原始 `photo` 的临时字段，不是后端接口字段。

当 `photo == null` 时，`azureBlobAbsolutePath(null)` 返回空字符串，因此列表可能表现为：

```json
{
  "photo": null,
  "photoAsbUrl": ""
}
```

## 一、上传返回路径差异

旧 SMMX 的 `AzureBlobProxy.uploadBlob` 上传后调用 `getRelativePath(...)`：

```text
smmx-vehicle-logistics/images.jpg
```

全球整车的旧 `AzureBlobProxy.uploadBlob` 直接返回 `blockBlobURL.toString()`：

```text
https://<blob-domain>/global-vehicle-logistics/images.jpg
```

旧 MX 前端保存时使用：

```js
photo: it.photo && it.photo.indexOf("http") == 0
  ? it.photoCopy
  : it.photo
```

该逻辑依赖“新上传返回相对路径”。如果全球 MX 返回绝对 URL，新照片会被误判成旧图预览地址，最终回退到 `photoCopy`。

### 修复

全球 MX `uploadPhoto` 使用：

```java
azureBlobService.uploadPublicFileSourceNameV2(uploadFile, container)
```

V2 返回：

```text
global-vehicle-logistics/images.jpg
```

容器名不同属于服务归属差异，不影响相对路径约定。

## 二、Gate Out 列表查询映射遗漏

异常响应特征：

```json
{
  "inventoryVehicleNo": null,
  "photo": null,
  "photoAsbUrl": null,
  "photoName": null
}
```

全球原查询 `findAllWarehouseInventoryVehicleBy` 使用：

```xml
resultType="com.smil.globalvehicle.warehouse.infrastructure.dto.InventoryVehicleDto"
```

但字段不能自动对应：

```text
status            -> inventoryVehicleStatus
organization_uuid -> uuid
gout.dn_no        -> gateOutDto.dnNo
gout.photo        -> gateOutDto.photo
```

`gateOutDto` 是嵌套对象，普通 `resultType` 无法正确组装。后续按库存状态、仓库和 DN 筛选时无法匹配，导致 `inventoryVehicleNo` 及图片字段为空。

### 隔离修复

为了不改变全球市场原查询链路，新增 MX 专用实现：

- Mapper 方法：`findAllMxWarehouseInventoryVehicleBy`
- ResultMap：`MxInventoryVehicleDtoMap`
- MX 专用 SQL：使用该 ResultMap，并查询 `gout.photo_name AS outPhotoName`
- `MxWarehouseInventoryVehicleQueryController.pageList` 改调 MX 专用方法

全球原 `findAllWarehouseInventoryVehicleBy` 及其调用保持不变。

修复生效的直接证据：列表可以返回非空 `inventoryVehicleNo` 和 `gateOutNotes`。此时若 `photo` 仍为空，说明查询关联已恢复，但数据库照片字段尚未写入。

## 三、Gate Out Confirm 丢失图片字段

MX 页面从主页面 Confirm 进入时，`moreParams=true`，调用：

```text
/import/gateOutByComfirm
```

单独点击 Upload Photo 时，`moreParams=false`，上传后弹窗 Confirm 调用：

```text
/savePhoto
```

因此：

- Gate Out Confirm 中上传的图片，应由 `gateOutByComfirm` 随出库记录一起保存，不需要再调用 `savePhoto`。
- 已出库车辆后续单独修改图片，才需要调用 `savePhoto`。

全球 MX `gateOutByComfirm` 的请求转换本来会将 `GateFileComfirmVPO.photo/photoName` 放入 `GateOutDto`，但 `GateOutNoteFactory.gateOutNote(DnAllDto, GateOutDto)` 构建领域对象时曾漏掉：

```java
.photo(gateOutDto.getPhoto())
.photoName(gateOutDto.getPhotoName())
```

所以会出现：

```text
前端请求有 photo
-> GateOutDto 有 photo
-> Factory 构建时丢失
-> Gate Out 成功
-> warehouse_inventory_vehicle_gated_out_note.photo 仍为 null
```

## 四、photoName 完整可空链路

仅在 Factory 增加 `.photoName(...)` 不够，必须完整传递：

1. `GateOutNote` 增加可空 `String photoName`。
2. `GateOutNoteFactory` 从请求构建时设置 `photo/photoName`。
3. `GateOutNoteFactory` 从 `GatedOutNoteDao` 恢复时带回 `photoName`。
4. `GatedOutNoteDao.of()` 保存时设置 `.photoName(...)`。
5. `photoName` 排除在 `GateOutNote.equals/hashCode` 外，避免改变其他市场对象比较逻辑。

其他市场不传时保持 `null`，不改变 Gate Out 状态、日期、DN、库存或海关状态校验。

## 五、savePhoto 行为

`savePhoto` 请求格式：

```json
{
  "gateOutPhotoInfoVdos": [
    {
      "dnNo": "DN...",
      "photo": "global-vehicle-logistics/images.jpg",
      "photoName": "images.jpg"
    }
  ]
}
```

后端遇到以下任一情况会静默跳过该项：

```java
StringUtils.isBlank(obj.getDnNo()) || StringUtils.isBlank(obj.getPhoto())
```

因此排查单独上传时必须同时检查：是否发出 `savePhoto`、Payload 的 `dnNo/photo` 是否非空、更新 SQL 是否命中对应 `dn_no`。

## 六、海关状态报错与图片无关

错误：

```text
customs status is wrong！
```

来自 Gate Out 前的 `bonded` 校验，与图片上传、V2 或列表 Mapper 无关。MX 国内 DN 通常要求车辆主数据 `bonded=FREE`；需结合发货仓是否保税、发货/收货国家、DN 类型核对。

## 七、SAS 地址有效期与中文文件名问题

`photoAsbUrl` 是查询时根据 `photo` 动态生成的 Azure Blob SAS 临时地址，不应持久化或长期复用。URL 参数含义：

- `st`：开始生效时间，UTC。
- `se`：失效时间，UTC。
- `sp=r`：只读权限。

当前 `azureBlobAbsolutePath(relativePath)` 使用 300 秒参数生成 SAS，但实际观察到的链接有效窗口约为 20 分钟，具体窗口还受 Azure SDK 的时钟偏移处理影响。链接过期后，应重新请求 `/pagelist/gateOut` 获取新的 `photoAsbUrl`。

若 Azure 返回 `BlobNotFound`，且 URL 文件名形如：

```text
%25E5%25BE%25AE...
```

则不是 SAS 过期，而是中文文件名被重复 URL 编码。`%25` 表示被编码的百分号；实际文件名经过了两层编码。

根因位于公共 V2 返回逻辑：

```java
URL url = blockBlobURL.toURL();
String fullPath = url.getPath();
return fullPath.substring(1);
```

`URL#getPath()` 返回已经编码的相对路径；该值入库后，再交给 Azure SDK 生成 SAS 时，百分号会再次编码，最终定位到不存在的 Blob。

正确方向是让数据库始终保存未编码的原始相对路径：

```text
global-vehicle-logistics/微信图片_xxx.jpg
```

然后只在生成访问 URL 时由 Azure SDK 编码一次。

### 当前处理决策（2026-07-15）

暂不修改公共 `uploadBlobV2()`，原因是它还被 Warehouse、Invoice、Vehicle Series、Sales Information、Attachment 等多个模块调用，公共返回格式变更需要跨模块回归。

MX Gate Out 当前采用文件名约束规避：

```text
A-Z a-z 0-9 - _ .
```

示例：

```text
gate_out_LVIN20260713003.jpg
images_20260715_001.jpg
```

暂时避免中文、空格及 `+`、`%`、`#`、`?` 等特殊字符。纯英文安全文件名不会触发该二次编码问题。

历史数据不会因未来修改上传方法而自动变化：

- 已保存的英文相对路径继续正常使用。
- 已保存成 `%E5...` 的中文编码路径仍可能 `BlobNotFound`。
- 此类历史图片需要重新使用英文文件名上传，或后续执行经过评估的定向数据修复。

## 最终验证清单

1. `uploadPhoto` 返回的 `attachmentUrl` 不含协议和域名。
2. Gate Out Confirm 请求中包含相对路径 `photo` 和可空 `photoName`。
3. Confirm 成功后，出库记录表 `photo/photo_name` 正确写入。
4. 已出库后单独上传时，前端调用 `savePhoto`，且 `dnNo/photo` 非空。
5. MX 列表返回非空 `inventoryVehicleNo`。
6. 列表 `photo` 为相对路径，`photoAsbUrl` 为可访问完整地址，`photoName` 与上传文件名一致。
7. 回归无照片的全球市场 Gate Out，确认 `photoName=null` 不改变原流程。
8. 回归 Gate Out Cancel、已出库查询和照片批量下载。
9. 当前阶段上传文件名仅使用英文、数字、短横线、下划线和扩展名点号。
10. 若打不开图片：先按 `se` 判断 SAS 是否过期；若响应为 `BlobNotFound`，再检查路径是否含 `%25E5...` 等二次编码特征。

## 代码定位

- MX Controller：`MxWarehouseInventoryVehicleQueryController`
- 应用服务：`InventoryVehicleApplicationService.mxGateOutFileImport/gateOutByDnAll/savePhoto`
- 图片领域工厂：`GateOutNoteFactory`
- 图片领域对象：`GateOutNote`
- 出库 DAO：`GatedOutNoteDao`
- MX 专用查询：`InventoryVehicleMapper.findAllMxWarehouseInventoryVehicleBy`
- Azure 相对路径：`GlobalVehicleAzureBlobService.uploadPublicFileSourceNameV2`
- 旧 MX 前端：`dcs-smmx-vehicle-client/.../gateOut.vue`、`gateOutDialog.vue`

## 验证状态

- 代码差异检查：`git diff --check` 通过。
- Mapper XML：结构解析通过。
- Maven 完整编译未完成：本地缺少 `dcs-excel-bean` 与 SAP JCo 私有依赖；不能将此项表述为编译已通过。
