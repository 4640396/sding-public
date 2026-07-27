---
title: "墨西哥 PO Internal Review 审批状态回退排查"
type: log
status: reference
created: 2026-07-13
updated: 2026-07-13
sensitivity: internal
source:
project: DCS
repositories:
  - dcs-global-vehicle
  - dcs-smmx-vehicle
---
# 墨西哥 PO Internal Review 审批状态回退排查

> [!warning] 方案状态
> 本文记录的 `SUBMITTED` 监听条件与 `purchaseInternalApprove()` 状态分支曾在本地试改，但在联调中发现会改变现有 GDS 链路，已于 2026-07-13 全部撤回。当前代码未保留这两处修改；本文仅作为问题分析线索，不能直接作为最终实施方案。

## 现象

墨西哥采购单在 `main/po-internal-review` 页面批准时调用：

`POST /order/internalApproved/po/mx/approve`

请求只包含采购订单号，接口返回：

`order status should be APPROVED!`

报错发生在批准后的整车订单初始化阶段，不是请求参数校验，也不是要求调用接口前订单状态已经为 `APPROVED`。

## 业务规则

`PO Internal Review` 开关控制墨西哥采购单是否先经过人工内部审核：

- 开关关闭：采购单提交后继续走全球原有流程；
- 开关开启：提交时将 `reviewStatus` 设置为 `INTERNAL_REVIEWING`，暂停后续全球流程，等待 `main/po-internal-review` 页面处理；
- 页面批准后：应恢复提交时暂停的全球流程，并执行最终批准后的处理。

开关配置维度：`po_internal_review`。

## 根因

页面批准后，`OrderApplicationService.internalApproval(PurchaseTradeOrderApprovalCommand)` 执行：

1. 校验 `reviewStatus == INTERNAL_REVIEWING`；
2. 原实现中的 `purchaseInternalApprove()` 无论合同是否需要外部审批，都将订单状态设置为 `APPROVED`；
3. 保存订单；
4. 发送 `OrderSubmitEvent`，恢复提交时暂停的价格包、合同、日志、邮件等全球流程；
5. 发送 `OrderApprovalEvent`，初始化整车订单及批准后业务。

该实现同时存在两个问题：

1. `purchaseInternalApprove()` 忽略 `orderApprovalSetting.isNeedApproval()`，导致需要 GDS 审批的订单被提前设置为最终 `APPROVED`；
2. `OrderSubmitEvent` 的 SMIL 自动内部审批监听器原来只判断 seller 是否为 SMIL，没有判断订单当前状态：

`seller == SMIL → internalApproval()`

本次订单 seller 对应 SMIL。监听器将已经是 `APPROVED` 的订单再次执行内部审批，导致状态回退：

`APPROVED → INTERNAL_APPROVED`

增加 `SUBMITTED` 判断后，重复自动内部审批被阻止，但也暴露了第一个问题：`/order/approval/gds` 明确要求回调前订单必须为 `INTERNAL_APPROVED`，提前写成 `APPROVED` 会返回 `order status should be Internal Approved!`。

## 与墨西哥老代码的差异

老墨西哥流程中，普通采购单审批方法不会先把订单直接设置成 `APPROVED`，而是发送 `OrderSubmitEvent`，由 SMIL 自动内部审批继续推进：

`SUBMITTED → OrderSubmitEvent → INTERNAL_APPROVED → 后续审批 → APPROVED`

迁移到 global 后，新流程增加了 `purchaseInternalApprove()`，页面人工批准后订单已经成为 `APPROVED`，但仍需要发送 `OrderSubmitEvent` 恢复此前暂停的全球提交业务。原监听器没有状态前置条件，因而产生重复审批和状态回退。

## 修改方案

> 以下为排查期间验证过但已经撤回的候选方案，不代表最终实施结论。

保留采购审批方法中的 `OrderSubmitEvent`，避免丢失价格包、合同、提交日志、邮件等监听逻辑。

修复包含两部分：

1. 约束 SMIL 自动内部审批监听器：必须同时满足 seller 为 SMIL，并且订单状态为 `SUBMITTED`；
2. `purchaseInternalApprove()` 按销售合同审批配置决定下一状态：需要外部审批时进入 `INTERNAL_APPROVED` 并发送第三方审批，不需要时直接进入 `APPROVED`。

修改文件：

`dcs-service/src/main/java/com/smil/globalvehicle/tradeorder/application/eventhandler/TradeOrderEventHandler.java`

修改后的判断：

```java
if (GlobalVehicleConstants.SMIL_SALES_COMPANY_NO.equals(tradeOrder.getSeller().getSalesCompanyNo())
        && TradeOrderStatusEnum.SUBMITTED.equals(tradeOrder.getOrderStatus())) {
    tradeOrder.internalApproval(tradeOrder.getShipper());
    orderRepository.save(tradeOrder);
    DomainEventPublisher.instance().publish(
            new OrderInternalApprovalEvent(tradeOrder, new ArrayList<>()));
}
```

`purchaseInternalApprove()` 的状态分支：

```java
if (this.orderApprovalSetting.isNeedApproval()) {
    this.setOrderStatus(TradeOrderStatusEnum.INTERNAL_APPROVED);
    this.setReviewStatus(TradeOrderReviewStatusEnum.INTERNAL_APPROVED);
    this.setSendToThird();
} else {
    this.setOrderStatus(TradeOrderStatusEnum.APPROVED);
    this.setReviewStatus(TradeOrderReviewStatusEnum.APPROVED);
}
```

## 修改后的流程

PO Internal Review 开启：

合同需要 GDS 审批：

`提交 → INTERNAL_REVIEWING → 页面批准 → INTERNAL_APPROVED → 恢复提交业务并发送第三方审批 → GDS 成功回调 → APPROVED → 创建整车订单`

合同不需要 GDS 审批：

`提交 → INTERNAL_REVIEWING → 页面批准 → APPROVED → 恢复提交业务 → OrderApprovalEvent → 创建整车订单`

普通 SMIL 订单：

`提交 → SUBMITTED → OrderSubmitEvent → 命中 SMIL + SUBMITTED → INTERNAL_APPROVED`

## 影响分析

- 非 SMIL seller：原本不会进入监听器，不受影响；
- 正常处于 `SUBMITTED` 的 SMIL 订单：仍执行自动内部审批，原流程不变；
- 已经 `APPROVED`、`INTERNAL_APPROVED`、已拒绝或其他非提交态订单：不再被重复内部审批；
- 墨西哥 PO Internal Review 页面批准：保留提交事件的其他业务，并按合同配置决定等待 GDS 或直接最终批准；
- `/order/approval/gds`：需要外部审批的订单在回调前保持 `INTERNAL_APPROVED`，成功回调后才进入 `APPROVED`。

新增判断表达了状态机约束：自动内部审批只允许 `SUBMITTED → INTERNAL_APPROVED`，不允许已批准订单倒退。

## 回归清单

1. 墨西哥采购单，PO Internal Review 开启且合同需要审批：页面批准后为 `INTERNAL_APPROVED`，GDS 成功回调后为 `APPROVED` 并创建整车订单；
2. 墨西哥采购单，PO Internal Review 开启且合同不需要审批：页面批准后直接为 `APPROVED` 并创建整车订单；
3. 墨西哥采购单，PO Internal Review 开启，页面拒绝：进入拒绝状态且不创建整车订单；
4. 墨西哥采购单，PO Internal Review 关闭：继续走全球原流程；
5. 普通 SMIL 销售单提交：仍能从 `SUBMITTED` 自动进入 `INTERNAL_APPROVED`；
6. 非 SMIL 市场订单提交：行为不变；
7. 检查价格包、合同、提交日志和邮件等 `OrderSubmitEvent` 监听业务仍被执行。

## 验证状态

- 已通过当前 global 与 SMMX 老代码对比确认调用链和状态变化；
- 曾在 `dcs-global-vehicle` 工作区加入 `SUBMITTED` 状态判断及合同审批状态分支，联调发现会影响现有 GDS 链路后已全部还原；
- Maven 编译因本机私有依赖缺失及 `C:\maven-repo` 写权限问题未完成，未发现由本次 Java 修改产生的编译错误，但仍需在完整构建环境执行编译和上述回归测试。
