---
title: "03 采购价格与 GDS 同步排查"
type: log
status: reference
created: 2026-07-11
updated: 2026-07-11
sensitivity: internal
source:
project: DCS
---
# 采购价格与 GDS 同步排查

## Price Not Found 查价顺序

系统依次尝试：

1. 客户价格包；
2. 国家价格包；
3. 客户采购价；
4. 国家采购价。

## 价格匹配维度

- `pricing_party = sellerUuid`；
- 客户级 `pricing_domain = buyer customer_no`，不是 buyer UUID；
- 国家级 `pricing_domain = countryCode`；
- `currency`；
- `price_type`；
- 有效日期；
- 物料编码；
- 价格包满足 `used_qty < max_qty`；
- 请求传入年份时还会匹配年份。

本次确认的主要原因是订单币种与价格币种不一致。UAT 墨西哥采购价格使用 `MXN`。

## 价格页面

查询接口：

`POST /dcs-global-api/price/purchase/pagelist`

页面数据来自：

- `price_fob`
- `price_freight`
- `price_insurance`

## GDS 同步

真实同步：

`POST /dcs-global-api/price/purchase/syncPriceFromGDS`

UAT 模拟写入：

`POST /dcs-global-api/price-ops/uploadGDSPrice`

```json
[
  {
    "brandCode": "MG",
    "vehicleMaterialCode": "MCL2LC13K1CMBC",
    "countryCode": "MX",
    "fobPurPrice": "10000.00",
    "freight": "100.00",
    "insurance": "100.00",
    "currency": "MXN",
    "validFromTime": "2026-01-01",
    "validEndTime": "2026-12-31"
  }
]
```

模拟接口会写入三张价格表，定价方固定为：

`saicorga-niza-tion-5000-globalveh000`

注意：模拟接口会校验物料是否存在于 SMIL/5000 组织。



