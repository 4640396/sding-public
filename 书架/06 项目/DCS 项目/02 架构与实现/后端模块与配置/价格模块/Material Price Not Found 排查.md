---
title: "Material Price Not Found 排查"
type: troubleshooting
status: verified
created: 2026-07-11
updated: 2026-07-11
sensitivity: internal
source:
project: DCS
---
# Material Price Not Found 排查

## 现象

订单物料查价或提交订单时，接口返回类似：

```text
Material: MEL8SG19ECLLMD Price Not Found!
```

这个错误不代表物料不存在，而是当前查询条件下没有匹配到可用价格。

## 典型接口

MX orderVehicleMaterial：

```text
/order/salesOrderQuery/mx/orderVehicleMaterial
```

示例请求：

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

大致调用链：

```text
TomSalesOrderQueryController
-> orderApplicationService.smmxGetOrderMaterial(query)
-> OrderHandleService.getMaterialPrice(...)
-> MaterialPriceQueryApplicationService.queryMaterialPrice(...)
-> Optional.empty
-> throw priceNotFound
```

## 实际查价参数

请求进入服务后，大致会变成：

```text
sellerUuid      = pricing_party
buyer           = buyerUuid 对应的 customer_no，不是 buyerUuid 本身
materialCode    = vehicleMaterialCode
currency        = orderCurrency
incoterm        = incotermCode
countryCode     = countryCode
price_type      = orderType 转换后的价格类型，例如 orderType=1 -> NORMAL
query date      = 当天零点
```

注意：

- 定价方 `pricing_party` 使用页面传入的 `sellerUuid`。
- 客户维度 `pricing_domain` 使用 buyer 对应的 `customer_no`。
- 国家维度 `pricing_domain` 使用 `countryCode`，例如 `MX`。

## 价格匹配顺序

常见匹配顺序：

1. 客户维度 Price Package：`pricing_domain = buyer.customerNo`
2. 国家维度 Price Package：`pricing_domain = countryCode`
3. 客户维度 incoterm 价格
4. 国家维度 incoterm 价格

只要这些条件都没有命中，就会报 `Price Not Found`。

## CIF 价格逻辑

`CIF = Cost + Insurance + Freight`。

系统里 CIF 不是只查一个单价，而是三段价格相加：

```text
CIF Price = FOB Price + Freight Price + Insurance Price
```

对应表：

```text
price_fob
price_freight
price_insurance
```

如果走 CIF 三段价格，通常三段都要有同一套匹配条件。只补其中一张或两张，CIF 仍然可能查不到完整价格。

## 维护价格的两种方向

### 方式 1：维护 Price Package

常见字段需要匹配：

```text
pricing_party = sellerUuid
pricing_domain = buyer customerNo 或 countryCode
currency = orderCurrency
price_type = orderType 转换后的价格类型
material_code = vehicleMaterialCode
valid_from <= query date
valid_to >= query date
used_qty < max_qty
```

### 方式 2：维护 CIF 三段价格

`price_fob`、`price_freight`、`price_insurance` 都要能按 seller、material、currency、价格类型、有效期等条件命中。

## submit 已有订单时的特殊点

接口：

```text
/order/salesOrder/mx/submit
```

如果请求里有已有订单号，例如：

```json
{
  "tradeOrderNo": "TOALLL2606160021",
  "sellerUuid": "3ca88d2d-ffe7-44eb-aa3b-24748a1949dd"
}
```

提交已有订单时，代码会走更新逻辑，先从数据库查原订单，再用原订单里的 seller / buyer / brand 重新查价。

也就是说 submit 查价用的是：

```java
order.getSeller()
order.getBuyer()
order.getBrand()
```

不一定是请求 body 里传入的 `sellerUuid`。

## 排查 SQL

### 查 buyer customerNo

```sql
select uuid, customer_no, name
from md_organization
where uuid = 'BUYER_UUID_HERE';
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
where trade_order_no = 'TRADE_ORDER_NO_HERE';
```

### 查国家/交付地址

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
where ppi.material_code = 'MATERIAL_CODE_HERE'
  and pp.pricing_party = 'SELLER_UUID_HERE'
  and pp.pricing_domain in ('BUYER_CUSTOMER_NO_HERE', 'COUNTRY_CODE_HERE')
  and pp.currency = 'CURRENCY_HERE'
  and pp.price_type = 1
  and pp.valid_from <= current_date
  and pp.valid_to >= current_date
  and pp.used_qty < pp.max_qty;
```

### 查 CIF 三段价格

```sql
select *
from price_fob
where pricing_party = 'SELLER_UUID_HERE'
  and material_code = 'MATERIAL_CODE_HERE'
  and currency = 'CURRENCY_HERE'
  and price_type = 1;

select *
from price_freight
where pricing_party = 'SELLER_UUID_HERE'
  and material_code = 'MATERIAL_CODE_HERE'
  and currency = 'CURRENCY_HERE';

select *
from price_insurance
where pricing_party = 'SELLER_UUID_HERE'
  and material_code = 'MATERIAL_CODE_HERE'
  and currency = 'CURRENCY_HERE';
```

## 隐蔽风险：PricePackageMapper 字段映射

如果 MyBatis 没开 `mapUnderscoreToCamelCase`，`select t1.*` 返回的字段：

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

结果可能变成：

```java
usedQty = 0
maxQty = 0
available() => 0 < 0 => false
```

于是 SQL 查到了价格包，但代码认为不可用。

建议 `PricePackageMapper.findOneBy` 使用显式 alias：

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

## 相关对话记录

- [[../../../05 历史与复盘/对话排查记录/2026-07-07 dcs-global-vehicle 对话拆解/04 订单价格与 Price Package 排查|2026-07-07 订单价格与 Price Package 排查]]






