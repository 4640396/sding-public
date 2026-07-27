---
title: "2026-07-09 项目 迁移墨西哥PurchaseInvoice到全球"
type: change-record
status: verified
created: 2026-07-09
updated: 2026-07-11
sensitivity: internal
source:
project: DCS
---
# 迁移墨西哥PurchaseInvoice到全球

## 需求标题

把墨西哥的main/purchase-invoice 页面的功能，迁移到全球。只迁移后端

## 来源

- 来源对话：
- 提出人：
- 时间：
- 原始材料：

## 项目

- [x] DCS
- [ ] IoT
- [ ] 其他：

## 当前状态

- [x] 收集
- [ ] 已进入工作台
- [ ] 分析中
- [ ] 开发中
- [ ] 待验证
- [ ] 已完成
- [ ] 已归档

## 需求描述

可以先看一下墨西哥前端的代码中的路由，找出purchase-invoice对应的接口有哪些。然后在去看墨西哥的后端代码的接口清单。然后在去全球的后端进行比对，如果全球有了。可以把墨西哥复制过来，在path里面增加一个mx路径

## 背景与现象

- 当前表现：
- 期望表现：
- 触发入口：
- 相关页面 / 接口 / 模块：

## 影响范围

- 墨西哥前端：`dcs-smmx-vehicle-client`
- 墨西哥后端：`dcs-smmx-vehicle`
- 全球前端：`dcs-global-vehicle-client`
- 全球后端：`dcs-global-vehicle`
- 前端：
- 后端：
- 数据库：
- 配置 / 部署：
- 权限 / 组织 / 数据：

## 待确认问题

- [ ] 
- [ ] 
- [ ] 

## 处理计划

1. 
2. 
3. 

## 处理记录

### YYYY-MM-DD

- 做了什么：
- 发现什么：
- 下一步：

## 验证方式

- 本地验证：
- 接口验证：
- 页面验证：
- SQL / 数据验证：
- 回归影响：

## 最终结论

这里写最后结果、原因、修改点和注意事项。

## 沉淀位置

- 项目笔记：
- 排查迷宫：
- 卡片盒：
- 归档：


## 2026-07-09 处理结果

已按“复制墨西哥 controller 到 global，并改成 `/mx/ic/invoice` 前缀”的方案落地。

### 最终需求口径

- 不直接改 global 现有 `IcInvoiceQueryController`，避免影响原 `/ic/invoice`。
- 新增 `MxIcInvoiceQueryController`，专门承接墨西哥 PurchaseInvoice 页面。
- 只迁移页面实际使用的接口。

### 已迁移接口

| 功能 | 新接口 |
| --- | --- |
| Purchase Invoice 分页 | `POST /mx/ic/invoice/purchase/pagelist` |
| Purchase Invoice Excel 下载 | `POST /mx/ic/invoice/purchase/download` |
| 发票预览 | `GET /mx/ic/invoice/preview/{invoiceNo}` |
| 发票明细 Excel 下载 | `GET /mx/ic/invoice/download/{invoiceNo}` |
| CIPL / BL 照片文件下载 | `POST /mx/ic/invoice/download/photoFiles` |

### 代码变更

- 新增 `MxIcInvoiceQueryController.java`。
- `GVInterfaceUrlAuthorityGroupEnum` 的 `PURCHASE_INVOICE` 权限组补充 `/mx/ic/invoice/**` 接口。
- `InvoiceMapper.mxQueryPurchaseAll` 补齐 `INVOICE_BUYER` 数据权限，alias 使用 global SQL 中实际存在的 `t3`。

### 底层链路确认

`MxIcInvoiceQueryController` -> `InvoiceQueryService.purchasePagelist(MxInvoiceQueryVDO)` -> `InvoiceMapper.mxQueryPurchaseAll` -> `mappers/ic/InvoiceMapper.xml::mxQueryPurchaseAll`。

global 已有墨西哥专用 `MxInvoiceQueryVDO`、`MxInvoicePurchaseVO`、`mxQueryPurchaseAll` SQL，本次没有新增底层 SQL，只补齐数据权限。

### 验证记录

- `git diff --check` 通过。
- `mvn -pl dcs-service -DskipTests compile` 被私有/商业依赖解析阻塞，缺少 `aspose-cells`、`dcs-excel-bean`、`sapjco3` 等依赖，未进入有效 javac 校验。

### 沉淀位置

- 详见：`06 项目/DCS 项目/05 历史与复盘/对话排查记录/2026-07 DCS 对话沉淀.md`


