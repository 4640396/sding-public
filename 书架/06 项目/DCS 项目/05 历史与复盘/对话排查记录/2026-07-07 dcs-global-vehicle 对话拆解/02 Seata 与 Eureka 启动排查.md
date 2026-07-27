---
title: "02 Seata 与 Eureka 启动排查"
type: log
status: reference
created: 2026-07-07
updated: 2026-07-11
sensitivity: internal
source:
project: DCS
---
# Seata 与 Eureka 启动排查

## 典型报错

```text
io.seata.common.exception.FrameworkException: No available service
no available service 'SEATA-SERVICE' found, please make sure registry config correct
```

启动日志里常见：

```text
Registered Applications size is zero : true
Getting all instance registry info from the eureka server
The response status is 200
no available service 'SEATA-SERVICE' found
```

## 核心判断

这不是开票业务校验失败，而是 Seata 全局事务基础设施不可用。

UK 开票链路里 `InvoiceApplicationService.createInvoice(...)` 有：

```java
@GlobalTransactional(rollbackFor = Exception.class, timeoutMills = 300000)
```

所以实际调用链是：

```text
MxIcInvoiceController.ukCreateByVin
-> InvoiceApplicationService.createUkInvoiceByVIN
-> InvoiceApplicationService.createInvoice
-> @GlobalTransactional 开启 Seata 全局事务
-> Seata TM 连接 TC
-> 找不到可用 TC 服务
-> 业务代码还没真正执行就失败
```

## Seata 查找 TC 的链路

Seata 客户端不是只看 Eureka 有没有 `SEATA-SERVICE`，它先要把事务组映射到服务名：

```text
tx-service-group
-> seata.service.vgroup-mapping.<tx-service-group>
-> SEATA-SERVICE
-> Eureka 拉实例
-> Netty 连接 TC 8091
```

常见配置：

```yaml
seata:
  enabled: true
  application-id: ${spring.application.name}
  tx-service-group: ${spring.application.name}
  registry:
    type: eureka
    eureka:
      service-url: ${eureka.client.serviceUrl.defaultZone}
  service:
    vgroup-mapping:
      dcs-service-global-vehicle: SEATA-SERVICE
```

## 本次 SIT 的真实根因

SIT 实际为：

```properties
spring.application.name = dcs-service-smmx-vehicle
seata.tx-service-group = ${spring.application.name}
```

运行时事务组变成：

```text
dcs-service-smmx-vehicle
```

但 Apollo 里只有：

```properties
seata.service.vgroup-mapping.dcs-service-global-vehicle = SEATA-SERVICE
```

Seata 会找：

```properties
seata.service.vgroup-mapping.dcs-service-smmx-vehicle
```

没有找到映射，最终报 `No available service`。

## 推荐修复

在 SIT Apollo 补：

```properties
seata.service.vgroup-mapping.dcs-service-smmx-vehicle = SEATA-SERVICE
```

链路变成：

```text
tx-service-group: dcs-service-smmx-vehicle
-> vgroup-mapping.dcs-service-smmx-vehicle
-> SEATA-SERVICE
-> Eureka 中的 SEATA-SERVICE:8091
```

修改后：

- 确认 Apollo 已发布。
- 服务最好重启。
- 如果仍报错，检查应用机器到 `172.29.10.8:8091` 的网络连通性。

## Eureka 指到本机的坑

有一次配置里发现：

```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: http://127.0.0.1:8761/eureka/
```

这会导致业务服务查本机空 Eureka，于是：

```text
Registered Applications size is zero : true
no available service 'SEATA-SERVICE'
```

应改成当前环境真实 Eureka，例如：

```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: http://172.29.10.10:8761/eureka/
```

## 启动成功不等于依赖正常

看到：

```text
############ 启动成功 #################
```

只能说明 Spring Boot 主体启动完成，不代表所有依赖都可用。

当时同时存在：

- Eureka 返回 200，但注册表为空。
- Seata 找不到 `SEATA-SERVICE`。
- Redis 获取连接失败。
- RabbitMQ 正在连接。

这属于“带故障启动”。依赖 Redis、Seata、服务发现的接口仍可能失败。

## Eureka 自我保护模式

Eureka 页面提示：

```text
EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP WHEN THEY'RE NOT.
RENEWALS ARE LESSER THAN THRESHOLD AND HENCE THE INSTANCES ARE NOT BEING EXPIRED JUST TO BE SAFE.
DS Replicas
```

含义：Eureka 收到的心跳续约数量低于阈值，进入自我保护模式，暂时不剔除失联实例。

常见原因：

- 大量服务重启或停止。
- 服务与 Eureka 网络不稳定。
- 服务注册到另一套 Eureka。
- Eureka Server 刚启动，阈值尚未稳定。
- 单机测试环境服务少，但阈值计算偏高。
- 服务被强杀，未正常注销。
- Eureka 集群配置错误，`DS Replicas` 为空。
- 机器时间不同步。

测试环境可临时关闭：

```yaml
eureka:
  server:
    enable-self-preservation: false
    eviction-interval-timer-in-ms: 5000
```

但这不能修复 Seata。Seata 仍要查服务是否注册、映射是否正确、网络是否能通。

## Eureka 显示服务 UP，但你没启动

如果 Eureka 显示：

```text
DCS-SERVICE-SMMX-VEHICLE
UP (1) - ea-sit-server02.internal.cloudapp.net:dcs-service-smmx-vehicle:8825
```

但你认为没有启动，常见原因：

1. 机器上有旧进程在跑，例如 Jenkins、systemd、Docker、nohup。
2. 另一个服务也配置了 `spring.application.name=dcs-service-smmx-vehicle`。
3. Eureka 旧注册信息还没过期。
4. Eureka 自我保护导致未剔除。
5. 看错环境。

排查命令：

```bash
ssh ea-sit-server02
ps -ef | grep dcs-service-smmx-vehicle
ps -ef | grep 8825
netstat -tlnp | grep 8825
lsof -i:8825
```

如果 8825 有 Java 进程，就是服务还在跑。如果没有进程但 Eureka 长时间显示 UP，看 instance detail 的 last renewal timestamp。

## 读音

`Seata` 一般读作“西塔”，英文近似 `SEE-tah`。







