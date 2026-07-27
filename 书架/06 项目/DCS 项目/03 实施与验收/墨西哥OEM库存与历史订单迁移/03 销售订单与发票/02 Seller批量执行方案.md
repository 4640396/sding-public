---
title: MX OEM STOCK Seller销售订单与发票批量执行方案
type: runbook
status: accepted
created: 2026-07-18
updated: 2026-07-25
sensitivity: internal
project: DCS
sources:
  - "[[../01 期初库存入库/03 批量导入执行方案]]"
  - "[[../01 期初库存入库/05 UAT客户号切换为270165决策]]"
  - "C:\\works\\initdata\\Order\\经销商采购导入模版.xlsx"
  - "dcs-global-vehicle OpsSmzaOrderDataInitializationController"
  - "dcs-global-vehicle OpsOrderDataInitializationController"
  - "dcs-global-vehicle OpsSmzaDataInitService"
  - "dcs-global-vehicle OpsDataInitInvoice"
---

# MX OEM STOCK Seller销售订单与发票批量执行方案

> [!summary] 只看这里：销售阶段到底做什么
> 本阶段处理的是“墨西哥公司已经卖给经销商的历史车辆”，不是总部卖给墨西哥的 OEM 采购。对同一辆 VIN，要恢复三类事实：`Order=销售订单`、`Logistics=从墨西哥库存发给经销商并收车`、`Invoice=销售发票`。采购订单、GDS 发运和 Gate In 属于前置阶段，必须先完成，但不装进本阶段的三个销售 Sheet。

```text
前置阶段（另一个模板）
总部 → 墨西哥：OEM采购订单 → GDS发运 → Gate In → 墨西哥库存

本阶段（经销商采购导入模版.xlsx）
墨西哥 → 经销商：Order销售订单 → Logistics出库/收车 → Invoice销售发票
```

你现在只需做四步：

1. 把 [[01 Seller模板生产只读取数#1、导出 Order Sheet|1号 Order SQL]]、[[01 Seller模板生产只读取数#2、导出 Invoice Sheet|2号 Invoice SQL]]、[[01 Seller模板生产只读取数#3、导出 Logistics Sheet|3号 Logistics SQL]] 发给运维。
2. 运维交付 `01_Order.xlsx`、`02_Invoice.xlsx`、`03_Logistics.xlsx`。
3. 核对三个文件的 `(Order No, VIN)` 集合一致，再分别粘贴到模板同名 Sheet。
4. UAT 先导入销售订单与物流，验收成功后再导入销售发票。

0号 SQL 只是我们已经完成的取数口径验证，不需要随三条导出 SQL 发给运维；历史附录也不执行。

> [!important] 适用边界
> 本方案承接 [[../01 期初库存入库/03 批量导入执行方案]]，仅用于 UAT Seller 销售订单和发票扩批。对应 VIN 的采购、发运、Gate In 和库存验收全部通过后，才可进入本阶段。UAT 扩批通过不等于生产就绪，也不授权生产写入。

> [!note] 当前阶段：SQL已定稿，待Order日期复测和运维全量导出
> 2026-07-18 已在生产只读平台验证 0 号候选 SQL 能返回 `vehicle_order_status=8` 且 DN、Gate Out、Received、Invoice 均完整的 `scope_check=PASS` 数据。2号 Invoice 与修正版3号 Logistics 已完成样例复测；1号 Order 的字段已验证完整，但 `Order Date` 改为 `seller_order.create_time` 后尚缺一次结果复测。三份全量 Excel 也尚未导出、组装和验收，因此当前不标记为全量导入就绪。

> [!success] 1号 Order SQL 样例字段完整通过
> 2026-07-18 在生产只读平台执行 1 号 SQL，已观察到结果列与模板 Order Sheet 对齐；源导出样例的历史 `Sales Company No=E000` 只用于追溯，目标UAT/生产模板统一映射为 `L000`。订单号、Incoterm、订单日期、经销商、目的地、地址、VIN、订单类型、Sales Price、`Currency=MXN`、Material Code 及物料/内饰/外饰描述均有生产值。

> [!warning] Order Date已修正，需复测
> 对订单 `MXASIMX2503270028` 的生产核对显示：`pricing_date/submit_time/approve_date=2025-03-27`，但订单记录 `create_time=2025-03-11`，发票日期也是 `2025-03-11`；原 1 号 SQL 将定价日期误作 Order Date。运维版已修正为 `Order Date=seller_order.create_time`，此前 Order 日期样例结论被本条替代，1号需按修正版复测。DN Date使用下方“最终映射修正”，不再引用本次中间判断。

> [!warning] DN Date最终映射修正
> 生产复测证明 `lm_delivery_note.create_time` 同样存在晚于 Gate Out/Received 的后补时间，不能表示历史DN业务日期。全球 `OpsSmzaDataInitService` 只检查模板 DN Date 是否非空来触发DN创建，创建参数不消费该日期；历史事件日期由 Gate Out和Received字段写入。因此最终3号SQL使用可靠的 `dni.gate_out_time` 同时填充 DN Date 与 Gate Out Date，移除对 `dn.create_time` 的筛选。该规则不伪造另一个未知日期，并保证 `DN Date = Gate Out Date <= Received Date`。

> [!success] 3号 Logistics SQL 修正版样例通过
> 2026-07-18 在生产只读平台执行修正版 3 号 SQL，可见样例的 Order No、VIN、From Warehouse、Destination Short Name、DN Type 均有值，且全部满足 `DN Date = Gate Out Date <= Received Date`；样例仓库包括 `MZP`、`LRI`，DN Type为 `1`。该证据确认日期映射修正生效，但尚未证明完整导出行数、唯一性及三文件集合一致。

> [!success] 2号 Invoice SQL 样例字段通过
> 2026-07-18 在生产只读平台执行 2 号 SQL，已观察到 Order No、VIN、Invoice No、Invoice Date、Invoice Price 五列均有生产值，且样例订单 `MXASIMX2503270028` 的发票日期为 `2025-03-11`、价格为 `406822.45`。该证据仅确认样例字段，不代表完整导出行数、唯一性及三文件集合已验收。

并非所有 Gate In 库存都进入 Seller 阶段。仍未销售给 Dealer 的当前库存应在 Gate In 后收口，不生成 Seller 销售单或发票；本方案只处理生产中已经存在真实 `L000` Seller 销售订单、完整物流和有效发票的历史已售车辆，候选范围按 [[01 Seller模板生产只读取数#0、全量Seller销售订单候选]] 确定。

## 一、接口顺序

1. `POST /ops/smza-order/initData`：Seller 销售订单 Excel 转 JSON。
2. `POST /ops/smza-order/save`：Seller 销售订单导入。
3. 验收销售订单、车辆分配、DN、Gate Out 和收车状态。
4. `POST /ops/smza-order/initInvoiceInfo`：Seller 发票 Excel 转 JSON。
5. `POST /ops/order/invoice`：Seller 发票导入。
6. 发票批后验收和批次收口。

同一批次未验收完成不得进入下一步。

## 二、模板准备

源模板为 `$env:WORKS_ROOT\initdata\Order\经销商采购导入模版.xlsx`。每批先复制为 `<批次号>-seller.xlsx`，不得覆盖源模板；保持 Sheet 顺序、名称和表头不变：

| Sheet序号 | Sheet | 用途 |
|---:|---|---|
| 0 | `Order` | 销售订单及物料明细 |
| 1 | `Invoice` | VIN 级发票明细 |
| 2 | `Logistics` | VIN 级 DN、Gate Out 和收车信息 |

模板自带南非样例。墨西哥批次必须清除全部样例数据行，不得保留 `ZA10`、`ZA`、`ZAR`、南非经销商、地址或仓库值。

模板的真实数据统一按 [[01 Seller模板生产只读取数]] 从生产只读库导出：1号结果粘贴 `Order`，2号结果粘贴 `Invoice`，3号结果粘贴 `Logistics`。不得把模板参考行当作批量数据，也不得手工编造缺失字段。

### 运维全量导出交付

运维只需分别执行 [[01 Seller模板生产只读取数#1、导出 Order Sheet|1号 SQL]]、[[01 Seller模板生产只读取数#2、导出 Invoice Sheet|2号 SQL]]、[[01 Seller模板生产只读取数#3、导出 Logistics Sheet|3号 SQL]]，交付：

```text
01_Order.xlsx
02_Invoice.xlsx
03_Logistics.xlsx
```

三条 SQL 都是生产只读平铺查询，无 VIN/OEM 订单号占位符、无时间条件、无临时表；使用同一套完整候选条件：生产 `L000` Seller 已审批订单、车辆状态 8、有效 DN、Gate Out、Received、有效发票、物料/价格/地址完整，并且每个VIN唯一追溯到一个`5000 → 270165`的已审批OEM订单。运维不得修改条件，也不得只导出页面最大显示行数。

收到文件后先验收，不直接调用接口：三个文件各自 `(Order No, VIN)` 不重复，且三者集合完全一致；不一致时停止并保留原始导出文件，不手工补行或删行掩盖问题。

## 三、Order Sheet填报

| 列 | 填报要求 |
|---|---|
| Order No | 必填；Seller 销售订单号，同一订单各 VIN 行一致 |
| Sales Company No | 必填；填组织编码，不填 UUID。当前UAT和生产的墨西哥 Seller 统一为 `L000`；源导出的历史 `E000` 必须在生成目标模板时映射为 `L000` |
| Incoterm Code / Order Date | 必填；使用真实业务值和可解析的真实日期 |
| DELIVERY SHORT NAME | 模板有两个同名列；两列填相同且已在 UAT 存在的目的地短名称 |
| DEALER SAP NO | 必填；填实际经销商 SAP 编码。`270165` 是当前墨西哥组织客户号，不等同于下游经销商编号，不得直接套用 |
| Delivery Name / Country Code / ADDRESS | 必填；墨西哥目的地资料，Country Code 为 `MX` |
| VIN | 必填；一行一个 VIN，属于本批且 Gate In 已成功 |
| Order Type | 必填；按真实业务类型填写 |
| Sales Price / Currency | 必填；真实销售价和业务币种，不能沿用南非 `ZAR` |
| 物料及描述字段 | 必填；与 VIN 和已导入车辆物料一致 |

转换器按 `Order No + Material Code` 汇总，`qty` 等于 Excel 行数。因此同一 VIN 不得重复，也不得用一行代表多辆车。

## 四、Logistics Sheet填报

| 列 | 填报要求 |
|---|---|
| Order No / VIN | 必填；与 Order Sheet 一一对应 |
| DN Date | 必填；真实 DN 日期 |
| Gate Out Date | 必填；真实出库日期 |
| Received Date | 必填；真实经销商收车日期 |
| From Warehouse | 必填；对应 VIN Gate In 后的实际 UAT 库存仓库 |
| Destination Short Name | 必填；与 Order Sheet 目的地一致 |
| DN Type | 必填；使用业务确认值 |

> [!warning] 当前转换器限制
> `initData` 会直接转换 DN Date、Gate Out Date、Received Date，空单元格有空指针风险。若业务尚未真实发生 Gate Out 或收车，不得伪造日期；应停止该订单，或另行修改转换逻辑并重新验收。

## 五、销售订单批前检查

必须全部通过：

```text
Order 去重订单数 = 批次订单数
Order VIN 去重数 = Logistics VIN 去重数
每个 VIN 在两个 Sheet 中各出现一次
每个订单+物料的 Order 行数 = 预期 qty
所有 VIN 已完成采购导入和 Gate In
当前库存仓库与 From Warehouse 一致
Sales Company No 在当前 UAT 存在且为目标 Seller
DEALER SAP NO 是实际下游经销商并在当前 UAT 存在
目的地、地址、国家、经销商和仓库关系已核验
订单号、VIN、物料、价格和三个物流日期均来自真实业务数据
UAT 不存在同 Seller 销售订单号或同 VIN 重复销售关系
```

任一项失败，停止整批，不转 JSON。

## 六、销售订单 Excel转JSON

```text
POST /ops/smza-order/initData
Content-Type: multipart/form-data
file=<批次 Seller Excel>
path=<UAT 应用服务器可写目录，末尾保留路径分隔符>
```

输出 `<path>smzaSalesOrder.json`。检查：

```text
顶层订单数 = Order Sheet 去重 Order No 数
salesCompanyNo 为业务确认的组织编码，不能是 UUID
dealerSapNo 为实际经销商编码
items[].qty = 同订单同物料 Excel 行数
logistics 数 = 对应 VIN 数
JSON 去重 VIN 数 = 当前批次 VIN 数
订单、VIN、物料、价格、仓库、目的地和日期与 Excel 一致
```

## 七、Seller销售订单导入

```text
POST /ops/smza-order/save
Content-Type: application/json
Body=<smzaSalesOrder.json 完整数列>
```

保存请求时间、HTTP 状态、完整响应和日志 trace。成功条件：

```text
响应外层 success = true
data.successList 数量 = 批次订单数
data.duplicateIgnored 为空
data.failed 为空
```

外层 `success=true` 但 `failed` 或 `duplicateIgnored` 非空，仍判定整批失败，不得立即重试。

## 八、销售订单批后验收

```text
销售订单数 = 批次订单数
车辆销售明细数 = 批次 VIN 数
每个 VIN 只关联一个当前批次销售订单
Seller、Dealer、物料、数量、价格和币种正确
车辆从 From Warehouse 正确分配
DN 数量和 VIN 明细正确
Gate Out 与 Received 状态及日期符合真实业务事实
发票导入前，每个目标车辆订单 data_status = 7（Matched）
```

车辆订单不是 `Matched` 时，发票服务会拒绝处理。任一项未通过，停止发票步骤。

## 九、Invoice Sheet填报

| 列 | 填报要求 |
|---|---|
| Order No | 本批已成功导入并验收的 Seller 销售订单号 |
| VIN | 属于该订单，且车辆订单状态为 `Matched` |
| Invoice No | 真实 Seller 发票号 |
| Invoice Date | 真实、可解析的发票日期 |
| Invoice Price | 仅保留人工对账；当前转换器不读取，不会进入发票 JSON |

一行代表一个 VIN，`Order No + VIN` 不得重复。

## 十、发票 Excel转JSON

```text
POST /ops/smza-order/initInvoiceInfo
Content-Type: multipart/form-data
file=<批次 Seller Excel>
path=<接口要求传入，但当前代码不使用该值决定输出位置>
```

当前代码固定输出到 UAT 应用服务器 `E:\initdata\Order\invoice.json`，后一次转换可能覆盖前一次文件。转换前确认无人并发执行；生成后立即复制并命名为 `<批次号>-invoice.json`，并核对生成时间。

```text
JSON 行数 = Invoice Sheet 数据行数 = 当前批次 VIN 数
Order No + VIN 唯一
每个 VIN 属于对应 Seller 销售订单
Invoice No、Invoice Date 与 Excel 一致
所有目标车辆订单 data_status = 7（Matched）
```

## 十一、Seller发票导入

```text
POST /ops/order/invoice
Content-Type: application/json
Body=<<批次号>-invoice.json 完整数列>
```

成功条件：

```text
响应外层 success = true
data.successList 数量 = Invoice JSON 行数
data.duplicateIgnored 为空
data.failed 为空
```

任一 VIN 报 `need Matched status`，说明销售订单或车辆状态未满足前置条件，停止整批排查，不得改 JSON 绕过。

## 十二、发票批后验收

```text
发票记录数 = Invoice JSON 行数
每个 Order No + VIN 只存在一条本批有效发票
Invoice No、Invoice Date 与 Excel 和 JSON 一致
发票关联的销售订单、车辆和 Dealer 正确
接口结果与数据库实际结果一致
```

`Invoice Price` 未由当前接口导入，须确认它仅用于对账还是需要其他接口落库；未确认前不得声称价格已随发票导入。

## 十三、批次策略

- 先用 1 个已完成 Gate In 的完整销售订单做 Seller 单单验证。
- 单单的销售、物流和发票全部通过后，扩到 5 个完整订单。
- 连续两批通过后再扩到 20 个完整订单，不拆同一销售订单。
- 每批独立保存 Excel、两个 JSON、订单/VIN/发票清单、接口完整响应和验收结果。
- 当前批全部收口后才开始下一批。

## 十四、立即停止条件

- 模板仍含南非样例或墨西哥主数据未经确认。
- Sales Company No 使用 UUID、仍为历史源值 `E000`，或目标值不是当前确认的 `L000`。
- 将 `270165` 当作下游 Dealer SAP No，但业务未确认。
- 三个物流日期任一为空或被伪造。
- VIN 未完成 Gate In，或库存仓库与 From Warehouse 不一致。
- 同一上游OEM订单已在其他阶段创建，但当前历史销售VIN未包含在该订单首次上游导入中。
- Excel 与 JSON 的订单、VIN、qty、价格、物流或发票数量不一致。
- UAT 已存在同订单、VIN 销售关系或发票。
- 任一接口超时、`failed` 或 `duplicateIgnored` 非空。
- 发票前车辆订单不是 `Matched`。
- 固定输出的 `invoice.json` 来源批次、生成时间或内容无法确认。

## 十五、首批执行前需固化的业务输入

```text
Seller 销售订单号
实际 Dealer SAP No
目的地短名称、名称、地址
销售价格、币种、Incoterm、Order Type
DN Type 和实际 From Warehouse
真实 DN Date、Gate Out Date、Received Date
Invoice No、Invoice Date
Invoice Price 是否仅用于对账，还是另有落库要求
```

这些输入未固化时，操作顺序和验收口径已经明确，但数据批次尚未达到可执行状态。

## 十六、2026-07-18阶段收口

### 已确认

- 运维只执行 [[01 Seller模板生产只读取数]] 中的1、2、3号SQL；0号验证SQL及历史错误版本均不交运维。
- 三条SQL共用同一候选范围：`L000` Seller已审批、车辆状态8、有效DN、Gate Out、Received、有效发票、物料/价格/地址完整，并可追溯到 `5000 → 270165` OEM订单。
- 目标UAT和生产模板组织编码统一输出为 `L000`；历史源导出中的 `E000` 只保留为追溯证据。Dealer使用真实下游经销商编号，不使用`270165`替代。
- 2号 Invoice 样例已显示订单号、VIN、发票号、发票日期和发票价格均有生产值。
- 3号 Logistics 修正版样例已显示订单号、VIN、仓库、目的地和DN Type均有值，且满足 `DN Date = Gate Out Date <= Received Date`。
- 1号 Order 样例已确认模板所需字段有值；最终SQL已将 `Order Date` 从错误的 `pricing_date` 修正为 `seller_order.create_time`。

### 尚未确认

- 尚未重新执行修正版1号SQL确认样例订单 `MXASIMX2503270028` 的 `Order Date=2025-03-11`。
- 运维尚未交付 `01_Order.xlsx`、`02_Invoice.xlsx`、`03_Logistics.xlsx` 的完整全量结果。
- 尚未验证三份Excel各自 `(Order No, VIN)` 唯一，以及三份集合完全一致。
- 尚未完成模板组装、Excel转JSON、UAT拆批导入和批后数据库验收。

### 下一步唯一执行顺序

1. 在生产只读平台重新执行1号Order SQL，确认修正后的Order Date。
2. 将现行1、2、3号SQL原样交给运维，要求完整导出三个Excel，不受页面最大行数限制。
3. 验收三个Excel的行数、键唯一性和键集合一致性；不一致立即停止，不手工补删数据。
4. 复制源模板并清除参考数据，分别装入三个同名Sheet。
5. 按“单单 → 5单 → 20单”的顺序在UAT拆批执行，并逐批保留Excel、JSON、接口响应和数据库验收结果。
