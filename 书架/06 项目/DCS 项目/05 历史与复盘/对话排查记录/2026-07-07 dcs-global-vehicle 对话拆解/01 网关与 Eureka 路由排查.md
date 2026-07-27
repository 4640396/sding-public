---
title: "01 网关与 Eureka 路由排查"
type: log
status: reference
created: 2026-07-07
updated: 2026-07-11
sensitivity: internal
source:
project: DCS
---
# 网关与 Eureka 路由排查

## 场景

网关配置只有：

```properties
zuul.routes.dcs-service-global-vehicle.path=/dcs-global-api/**
```

Eureka 中看到：

```text
DCS-SERVICE-SMMX-VEHICLE
n/a (2) (2) DOWN (2)
- VPTIT10095.smc.saicmotor.com:dcs-service-smmx-vehicle:8825
- ea-sit-server02.internal.cloudapp.net:dcs-service-smmx-vehicle:8825
```

当时目标：为了验证 bug，临时让网关请求固定打到 `VPTIT10095.smc.saicmotor.com:8825`。

## 结论

可以强制，最简单方式是临时绕过 Eureka，直接配 URL：

```properties
zuul.routes.dcs-service-global-vehicle.path=/dcs-global-api/**
zuul.routes.dcs-service-global-vehicle.url=http://VPTIT10095.smc.saicmotor.com:8825
```

注意不要写成：

```properties
http://VPTIT10095.smc.saicmotor.com:dcs-service-smmx-vehicle:8825
```

Eureka 页面里的：

```text
VPTIT10095.smc.saicmotor.com:dcs-service-smmx-vehicle:8825
```

只是展示格式，接近：

```text
host:applicationName:port
```

真实 HTTP 地址只能是：

```text
http://VPTIT10095.smc.saicmotor.com:8825
```

## 只有 path 时的隐式 serviceId

如果没有显式配置 `serviceId`，只有：

```properties
zuul.routes.dcs-service-global-vehicle.path=/dcs-global-api/**
```

Zuul 会默认把 route key 当 serviceId，等价于：

```properties
zuul.routes.dcs-service-global-vehicle.serviceId=dcs-service-global-vehicle
```

这就容易和实际 Eureka 服务名混淆：

```text
route key: dcs-service-global-vehicle
Eureka 服务名: DCS-SERVICE-SMMX-VEHICLE
应用名: dcs-service-smmx-vehicle
```

排查路由时，要把这三个名字分开看。

## 保留 serviceId 的固定实例方式

如果原本使用：

```properties
zuul.routes.dcs-service-global-vehicle.path=/dcs-global-api/**
zuul.routes.dcs-service-global-vehicle.serviceId=DCS-SERVICE-SMMX-VEHICLE
```

可以通过 Ribbon 固定 server list：

```properties
DCS-SERVICE-SMMX-VEHICLE.ribbon.NIWSServerListClassName=com.netflix.loadbalancer.ConfigurationBasedServerList
DCS-SERVICE-SMMX-VEHICLE.ribbon.listOfServers=VPTIT10095.smc.saicmotor.com:8825
```

有些版本会看到：

```properties
ribbon.eureka.enabled=false
```

这个可能影响全局 Ribbon 行为，不建议为了单个服务轻易加。

## 生效与验证

配置要发在网关应用自己的配置里，不是业务后端服务配置里。

如果走 Apollo / 配置中心，发布后网关不一定立刻刷新。保险做法：

- 重启网关。
- 或调用网关 `/actuator/refresh` / `/refresh`。

验证方式：

- 看 `VPTIT10095` 目标机器日志是否收到请求。
- 临时打开 Zuul / Ribbon 相关日志，确认转发地址。
- 验证完成后恢复原来的 Eureka / serviceId 路由。

## 适用建议

- 临时验证指定机器 bug：用 `url=http://host:port` 最清楚。
- 长期运行：优先走 Eureka，确保实例状态 UP。
- 如果 Eureka 中实例是 DOWN，Ribbon 正常不会选择它。







