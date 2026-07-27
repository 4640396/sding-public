---
title: "05 鉴权、用户组织与 NPE"
type: log
status: reference
created: 2026-07-07
updated: 2026-07-11
sensitivity: internal
source:
project: DCS
---
# 鉴权、用户组织与 NPE

## Authorization 请求头非法

### 背景

调用接口时报错，怀疑是这里没有取到组织：

```java
String uuid = systemUserUtils.me().getOrganizationUUID().getUuid()
```

但截图中的真正报错是：

```text
Error: Invalid character in header content ["Authorization"]
```

### 结论

这不是后端 `organizationUUID` 没取到。这个错误发生在 Postman 发送请求之前，请求还没到后端。

最可能原因是 `Authorization` 的 value 里有非法字符：

- 换行符
- 回车
- 隐藏字符
- 前后空格
- token 被复制成多行

### 修复

1. 清空 Authorization value。
2. 重新粘贴成一整行。
3. 不带引号，不换行，不加多余空格。
4. 按系统要求决定是否加 `Bearer`。

裸 token：

```text
eyJhbGciOi...
```

Bearer token：

```text
Bearer eyJhbGciOi...
```

从项目测试代码看，很多地方直接传 token，没有加 `Bearer`。

修好 header 后，再调：

```text
GET /api/dcs-smmx-veh-api/system/me
```

看返回里有没有 `organizationUUID`。

## NPE：AmOrgCode.sysCustomerNo orgCode 为空

### 报错定位

NPE 出在：

```java
AmOrgCode.sysCustomerNo(AmOrgCode.java:19)
```

代码类似：

```java
public String sysCustomerNo(){
    if(orgCode.equalsIgnoreCase(MasterDataConstants.SMME_SYS_ORG_CODE)){
        return MasterDataConstants.SMME_CUSTOMER_NO;
    } else {
        return orgCode;
    }
}
```

当 `orgCode == null` 时，`orgCode.equalsIgnoreCase(...)` 直接 NPE。

### 调用链

```text
PageInfoInterceptor.preHandle()
-> SystemUserUtils.myInfo()
-> SystemUserApplicationService.findOrGenerateCurrentSystemUser()
-> 当前用户查不到，走自动创建
-> SystemUserFactory.newSystemUser()
-> new AmOrgCode(userInfo.getOrgCode())
-> AmOrgCode.sysCustomerNo()
-> orgCode 为 null，NPE
```

### 伴随日志

```text
app code do not belong to global vehicle!
```

这句来自 `SystemUserFactory`，只是打 error，没有中断流程。后续仍然拿 `userInfo.getOrgCode()` 创建用户，最终 NPE。

### 最可能原因

- 请求头/登录上下文里没有 `orgcode`。
- `appcode` 不是 Global Vehicle 的 appCode。
- 当前用户在系统用户表里不存在，触发自动创建逻辑。
- 如果用户已存在，可能不会走到 NPE。

### 排查

- 请求 header 是否有 `orgcode`、`ucode`、`appcode`。
- `appcode` 是否等于 `GlobalVehicleConstants.GLOBAL_VEHICLE_APP_CODE`。
- 当前 `userCode + orgCode` 是否已有系统用户记录。

### 代码建议

在 `AmOrgCode.sysCustomerNo()` 或 `SystemUserFactory.validate()` 对空 `orgCode` 抛业务异常，不要让用户信息缺失变成 NPE。







