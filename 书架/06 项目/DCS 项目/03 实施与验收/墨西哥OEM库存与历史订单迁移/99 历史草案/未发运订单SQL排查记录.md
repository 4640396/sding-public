---
title: 未发运订单SQL排查记录
type: investigation-record
status: deprecated
superseded_by: "[[../02 未发运订单/01 未发运订单迁移执行方案]]"
created: 2026-07-18
updated: 2026-07-19
sensitivity: internal
project: DCS
sources:
  - "[[../02 未发运订单/01 未发运订单迁移执行方案]]"
---

# 未发运订单SQL排查记录

> [!warning] 历史记录，不执行
> 本文保留2026-07-18形成正式全量方案前的排查过程。现行操作只使用[[../02 未发运订单/01 未发运订单迁移执行方案]]，不得从本文复制SQL交运维或执行导入。

## 一、被替代的方案

- 旧版固定单车POC物料`MCL2LC13K1CMBC`，不能覆盖全量订单，已废弃。
- 旧版只通过`VIN → GDS`判断发运，可能把当前车辆订单无VIN、但GDS已有成功发运的订单误判为未发运，已废弃。
- 旧版单条0号SQL同时连接车辆明细和GDS明细，在订单内产生明细乘积，生产执行长时间无结果，已拆解并最终被正式全量SQL替代。
- 0A、0B带订单号的抽样查询只用于形成和验证筛选口径，不是运维全量交付SQL。

## 二、0A抽样结果

0A按“订单+物料”检查：

```text
vehicle_order_count = material_qty
allocated_vin_count = 0
candidate_check = CANDIDATE:NO_VIN
```

生产只读执行后确实返回多条候选，证明当前生产存在“车辆订单已创建但VIN为空”的记录。但该结果只能说明无VIN，不能说明无GDS。

## 三、0B抽样结果

0B按`customer_order_no + e_type='S'`直接检查成功GDS，得到以下证据：

| 订单 | 结果 | 历史结论 |
|---|---|---|
| `MXASMIL2502060001` | 返回21条成功GDS物料记录 | 否决，不是未发运订单 |
| `MXASMIL2502070001` | 返回5条成功GDS物料记录 | 否决，不是未发运订单 |
| `MXASMIL25021740001` | 0B未返回 | 仅通过无GDS检查，后续因买方UUID不一致被否决 |

前两条订单同时满足“当前车辆订单VIN为空”和“历史GDS已有成功发运”，说明生产不同数据域之间存在关联或状态不一致。由此确立硬规则：不能只凭VIN为空判断未发运，正式全量SQL必须整单排除成功GDS。

后续对`MXASMIL25021740001`的组织一致性核对得到`STOP:BUYER_UUID_MISMATCH`。因此“无VIN+无GDS”仍不足以准入，正式SQL又增加了`customer_no + sales_company_no=L000 + buyer_uuid`一致性条件。

## 四、保留的技术结论

- 全量范围不固定物料、不增加时间条件。
- 生产卖方`5000`、生产买方`270165`、目标UAT买方`220202`。
- 订单必须已审批，且每个物料的车辆订单数等于物料qty。
- 整单全部车辆VIN为空，整单不存在`e_type='S'`成功GDS。
- Excel发运Sheet为空时，转换器行为仍需UAT验证：控制器直接遍历和分组第2个Sheet；若Excel工具返回`null`会发生空指针，返回空列表才会生成`deliveryItemVos=[]`。
- 保存服务源码支持空发运集合，只保存采购订单和空VIN车辆订单；这不代表转换接口和后续真实GDS链路已经验收。

## 五、未完成事项

- 正式全量SQL尚未由运维完整导出。
- 空发运Sheet转JSON尚未实测。
- UAT单单采购导入尚未完成。
- 已迁移订单承接后续真实或模拟GDS发运的链路尚未验收。
