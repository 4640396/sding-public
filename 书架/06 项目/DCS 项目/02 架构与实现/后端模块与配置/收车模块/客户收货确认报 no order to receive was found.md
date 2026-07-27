---
title: "客户收货确认报 no order to receive was found"
type: troubleshooting
status: verified
created: 2026-07-11
updated: 2026-07-11
sensitivity: internal
source:
project: DCS
---
# 客户收货确认报 no order to receive was found

## 现象

调用 `/mx/customerGoodsReceive/save` 做客户收货确认时，页面或日志提示：

```text
no order to receive was found. LVIN20260707002
LmApplicationBusinessException: {lm.goodsReceive.vin.notFoundOrder} LVIN20260707002
```

## 核心结论

这个异常不一定表示 VIN 不存在，更常见的含义是：

```text
当前登录组织查不到自己可收货的订单。
```

客户收货确认会使用当前登录用户的组织 UUID 作为收货组织去查订单。谁是订单的 `delivery_destination_organization_uuid`，谁才能做 customer goods receive。

`delivery destination` 可以理解为实际收货组织或交付目的组织。它和 `buyer`、`seller` 是不同字段；业务上 buyer 和 delivery destination 有时相同，但代码校验客户收货时主要看当前登录组织是否匹配可收货订单条件。

## 代码入口

入口接口：

```text
com.smil.globalvehicle.logistics.userinterface.restful.MxLmCustomerGoodsReceiveController.save
```

关键服务：

```text
com.smil.globalvehicle.logistics.application.service.CustomerReceiveService.goodsReceive
```

接口会把当前登录用户组织写入 command：

```java
command.setOrganizationUuid(SystemUserUtils.myInfo().getOrganizationUUID().getUuid());
```

后续按 VIN 和当前登录组织查订单，查不到时抛：

```java
throw new LmApplicationBusinessException("{lm.goodsReceive.vin.notFoundOrder}" + " " + vin);
```

## 查询条件

订单查询会受这些条件影响：

- VIN 必须存在于 `tom_vehicle_order`。
- `tom_vehicle_order.data_status` 必须在 `(4, 5, 7, 8)`。
- 当前登录组织必须匹配业务查询条件。
- 普通客户收货场景主要看 `tom_trade_order.delivery_destination_organization_uuid = 当前登录组织`。
- 如果当前组织是销售公司，代码会先处理 `source_code = '2'` 的 non-seller 分支，再处理 seller 分支。
- 查询还需要能关联到 `tom_trade_order`、`tom_trade_order_info`。

## 排查 SQL

先确认 VIN 是否存在：

```sql
select *
from tom_vehicle_order
where vin = 'LVIN20260707002';
```

再确认订单归属组织、收货目的组织和 source code：

```sql
select
  tvo.vehicle_order_no,
  tvo.trade_order_no,
  tvo.vin,
  tvo.data_status,
  tvo.dn_no,
  tto.seller_uuid,
  tto.buyer_uuid,
  tto.delivery_destination_organization_uuid,
  tto.source_code,
  ttoi.trade_order_no as has_trade_order_info
from tom_vehicle_order tvo
left join tom_trade_order tto
  on tvo.trade_order_no = tto.trade_order_no
left join tom_trade_order_info ttoi
  on tvo.trade_order_no = ttoi.trade_order_no
where tvo.vin = 'LVIN20260707002';
```

## 本次案例

VIN `LVIN20260707002` 在 `tom_vehicle_order` 中存在两条记录：

```text
VOALLL2607070001 / TOALLL2607070007 / data_status = 7 / source_code = 1
VOASML2607070001 / TOASML2607070001 / data_status = 8 / source_code = 2
```

两条订单的 `delivery_destination_organization_uuid` 都是：

```text
108cf037-b932-48c2-ae36-f001f279decd
```

因此客户收货需要使用该 delivery destination 组织对应的账号操作。如果使用 seller、销售公司或其他组织账号，就会因为当前登录组织不匹配而报 `no order to receive was found`。

## 相关对话记录

- [[../../../05 历史与复盘/对话排查记录/2026-07-07 dcs-global-vehicle 对话拆解/08 物流客户收货确认排查|2026-07-07 物流客户收货确认排查]]





