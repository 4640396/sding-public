---
title: "02 MDM 物料编码校验"
type: log
status: reference
created: 2026-07-11
updated: 2026-07-11
sensitivity: internal
source:
project: DCS
---
# MDM 物料编码校验

## 错误示例

`The vehicle material code info not exists.[LGRDSG49NCCDRT]`

## 含义

当前环境调用 MDM 后，返回的物料编码集合中不存在该编码。此时尚未进入本地车系校验和物料保存逻辑。

## 检查项

- MDM 是否已维护该物料编码；
- 编码大小写是否一致；
- 是否包含前后空格；
- UAT 是否连接正确的 MDM 环境；
- MDM `getAll()` 是否实际返回该记录。

## 相关实现

`vehicleMaterialRepository.findAllVehicleMaterial()` 最终调用 MDM 的 `vehicleClient.getAll()`。



