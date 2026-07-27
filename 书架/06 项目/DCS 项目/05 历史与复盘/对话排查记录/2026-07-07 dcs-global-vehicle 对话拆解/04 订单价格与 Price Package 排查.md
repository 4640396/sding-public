---
title: "04 订单价格与 Price Package 排查"
type: log
status: reference
created: 2026-07-07
updated: 2026-07-11
sensitivity: internal
source:
project: DCS
---
# 订单价格与 Price Package 排查

## 典型报错

接口返回：

```text
Material: MEL8SG19ECLLMD Price Not Found!
```

这个错误不是物料不存在，而是当前条件下物料价格没有匹配到。

## 接口：MX orderVehicleMaterial

接口：

```text
/order/salesOrderQuery/mx/orderVehicleMaterial
http://172.29.10.11/api/dcs-smmx-veh-api/order/salesOrderQuery/mx/orderVehicleMaterial
```

请求示例：

```json
{
  "brand": "MG",
  "buyerUuid": "a5dc4895-9c8f-4c46-8a09-c6b2b5c481ba",
  "orderCurrency": "CNY",
  "incotermCode": "CIF",
  "countryCode": "MX",
  "orderType": "1",
  "sellerUuid": "3ca88d2d-ffe7-44eb-aa3b-24748a1949dd",
  "vehicleMaterialCode": "MEL8SG19ECLLMD"
}
```

调用链：

```text
TomSalesOrderQueryController
-> /order/salesOrderQuery/mx/orderVehicleMaterial
-> orderApplicationService.smmxGetOrderMaterial(query)
-> OrderHandleService.getMaterialPrice(...)
-> MaterialPriceQueryApplicationService.queryMaterialPrice(...)
-> Optional.empty
-> throw priceNotFound
```

## 实际查价参数

大致会变成：

```text
sellerUuid      = 3ca88d2d-ffe7-44eb-aa3b-24748a1949dd
buyer           = buyerUuid 对应的 customerNo，不是 buyerUuid 本身
materialCode    = MEL8SG19ECLLMD
currency        = CNY
incoterm        = CIF
countryCode     = MX
price_type      = 1  // orderType=1 -> NORMAL
query date      = 今天零点
```

注意：

- 定价方 `pricing_party` 使用页面传入的 `sellerUuid`。
- 客户维度 `pricing_domain` 使用 buyer 对应的 `customer_no`。
- 国家维度 `pricing_domain` 使用 `countryCode=MX`。

## 价格匹配顺序

1. 客户维度 Price Package：`pricing_domain = buyer.customerNo`
2. 国家维度 Price Package：`pricing_domain = MX`
3. 客户维度 incoterm 价格
4. 国家维度 incoterm 价格

## CIF 的含义

`CIF = Cost + Insurance + Freight`。

系统里 CIF 不是只查一个单价，而是三段：

```text
CIF Price = FOB Price + Freight Price + Insurance Price
```

对应表：

```text
price_fob
price_freight
price_insurance
```

如果三段价格都空，就会报 `Price Not Found`。

## 维护价格的两种方式

### 方式 1：维护 Price Package

字段需要匹配：

```text
pricing_party = sellerUuid
pricing_domain = buyer customerNo 或 MX
currency = CNY / MXN
price_type = 1
material_code = MEL8SG19ECLLMD
valid_from <= current_date
valid_to >= current_date
used_qty < max_qty
```

### 方式 2：维护 CIF 三段价格

`price_fob`、`price_freight`、`price_insurance` 都要有同一套匹配条件。

只补其中一张或两张，CIF 还是查不到。

## 接口：MX submit 已有订单

接口：

```text
/order/salesOrder/mx/submit
http://172.29.10.11/api/dcs-smmx-veh-api/order/salesOrder/mx/submit
```

如果请求里有：

```json
"tradeOrderNo": "TOALLL2606160021",
"sellerUuid": "3ca88d2d-ffe7-44eb-aa3b-24748a1949dd"
```

提交已有订单时，代码走 `makeUpdateTradeOrder()`，会先从数据库查原订单，然后用原订单里的 seller / buyer / brand 重新查价。

所以 submit 查价用的是：

```java
order.getSeller()
order.getBuyer()
order.getBrand()
```

不是请求 body 里的 `sellerUuid`。

## 常用排查 SQL

### 查 buyer customerNo

```sql
select uuid, customer_no, name
from md_organization
where uuid = 'a5dc4895-9c8f-4c46-8a09-c6b2b5c481ba';
```

### 查订单真实 seller/buyer

```sql
select
  trade_order_no,
  seller_uuid,
  buyer_uuid,
  brand,
  order_currency,
  incoterm_code,
  delivery_destination,
  order_type
from tom_trade_order
where trade_order_no = 'TOALLL2606160021';
```

### 查 MX01 国家码

```sql
select
  customer_no,
  delivery_short_name,
  country_code
from md_organization_customer_delivery_address
where delivery_short_name = 'MX01';
```

### 查 Price Package 是否命中

```sql
select
  pp.price_package_uuid,
  pp.pricing_party,
  pp.pricing_domain,
  pp.pricing_domain_type,
  pp.price_type,
  pp.currency,
  pp.valid_from,
  pp.valid_to,
  pp.max_qty,
  pp.used_qty,
  ppi.material_code,
  ppi.transaction_price,
  ppi.delivery_price,
  ppi.colour_price,
  ppi.basic_price,
  ppi.year
from price_package pp
join price_package_item ppi on pp.price_package_uuid = ppi.price_package_uuid
where ppi.material_code = 'MEL8SG19ECLLMD'
  and pp.pricing_party = '3ca88d2d-ffe7-44eb-aa3b-24748a1949dd'
  and pp.pricing_domain = 'MX'
  and pp.pricing_domain_type = 1
  and pp.currency = 'MXN'
  and pp.price_type = 1
  and pp.valid_from <= current_date
  and pp.valid_to >= current_date
  and pp.used_qty < pp.max_qty;
```

### 查 CIF 三段价格

```sql
select *
from price_fob
where pricing_party = '3ca88d2d-ffe7-44eb-aa3b-24748a1949dd'
  and material_code = 'MEL8SG19ECLLMD'
  and currency = 'CNY'
  and price_type = 1;

select *
from price_freight
where pricing_party = '3ca88d2d-ffe7-44eb-aa3b-24748a1949dd'
  and material_code = 'MEL8SG19ECLLMD'
  and currency = 'CNY';

select *
from price_insurance
where pricing_party = '3ca88d2d-ffe7-44eb-aa3b-24748a1949dd'
  and material_code = 'MEL8SG19ECLLMD'
  and currency = 'CNY';
```

## PricePackageMapper 隐蔽风险

如果 MyBatis 没开 `mapUnderscoreToCamelCase`，`select t1.*` 返回的：

```text
price_package_uuid
max_qty
used_qty
valid_from
valid_to
```

可能映射不到 Java 字段：

```java
pricePackageUuid
maxQty
usedQty
validFrom
validTo
```

结果：

```java
usedQty = 0
maxQty = 0
available() => 0 < 0 => false
```

于是 SQL 查到了价格包，但代码认为不可用。

建议把 `PricePackageMapper.findOneBy` 改成显式 alias：

```sql
select
  t1.price_package_uuid as pricePackageUuid,
  t1.pricing_party as pricingParty,
  t1.pricing_domain as pricingDomain,
  t1.pricing_domain_type as pricingDomainType,
  t1.price_type as priceType,
  t1.max_qty as maxQty,
  t1.used_qty as usedQty,
  t1.currency as currency,
  t1.valid_from as validFrom,
  t1.valid_to as validTo,
  t1.create_time as createTime,
  t1.update_time as updateTime
```







