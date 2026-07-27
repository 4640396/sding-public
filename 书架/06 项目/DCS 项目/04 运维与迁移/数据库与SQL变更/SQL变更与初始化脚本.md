---
title: "SQL变更与初始化脚本"
type: runbook
status: verified
created: 2026-07-11
updated: 2026-07-25
sensitivity: internal
source:
project: DCS
---
# SQL 变更与初始化脚本

## global SQL 位置

```text
$env:WORKS_ROOT\dcs\dcs-global-vehicle\document\00_初始化SQL文件\
$env:WORKS_ROOT\dcs\dcs-global-vehicle\document\01_迭代管理\
$env:WORKS_ROOT\dcs\dcs-global-vehicle\document\04_SQL变更记录\
$env:WORKS_ROOT\dcs\dcs-global-vehicle\document\测试\主数据初始化数据\
$env:WORKS_ROOT\dcs\dcs-global-vehicle\dcs-test\SQL\develop_global_vehicle_junit.sql
```

近期重点 SQL：

```text
document/04_SQL变更记录/f-DCS_Vehicle-Q2-Q4-RFC2026-1377-VinLifeCycle.sql
document/04_SQL变更记录/feature-smmx-initData-20260518.sql
document/04_SQL变更记录/feature-smmx-initProject-20260518.sql
document/04_SQL变更记录/feature-W-G-REQ-105-oemStock-20260322.sql
document/04_SQL变更记录/hotfix-W-G-REQ-87-fixFBIError-20260310.sql
document/01_迭代管理/2026_Q2-Sprint2/f-S-G-REQ-15-MonthlyTarget-20260617.sql
document/墨西哥配置管理/01_apollo_global_20260527_init.txt
document/墨西哥配置管理/01_apollo_smmx_20260527_bak.txt
```

## SMMX SQL 位置

```text
$env:WORKS_ROOT\dcs\dcs-smmx-vehicle\document\00_初始化SQL文件\
$env:WORKS_ROOT\dcs\dcs-smmx-vehicle\document\01_迭代管理\
$env:WORKS_ROOT\dcs\dcs-smmx-vehicle\document\02_需求管理\
$env:WORKS_ROOT\dcs\dcs-smmx-vehicle\dcs-test\SQL\
```

## 执行 SQL 前检查

- 确认目标库：dev / sit / pre / prd。
- 确认脚本是否可重复执行，是否包含 `insert` 冲突、`alter` 重复字段、索引重复创建等风险。
- 对生产环境先备份或走变更流程。
- 脚本与代码分支要匹配，特别是 SMMX init、VIN Life Cycle、OEM Stock、Monthly Target 等近期功能。
- 如果脚本涉及配置中心，确认 Apollo 配置是否同步。

## 排查 SQL 问题时看什么

- 表是否存在，字段是否存在，字段类型是否和实体/mapper 匹配。
- 初始化数据是否缺失，尤其主数据、组织、车型、物料、配置类数据。
- 枚举值、状态值是否和前端字典一致。
- job 依赖的配置项是否已初始化。
- 外键或唯一键是否阻止清理/重复导入。

## 业务数据迁移

- [[墨西哥业务数据迁移导入流程|墨西哥业务数据迁移导入流程]]：seller / non-seller 模式下的采购订单、Gate In、销售订单与发票导入流程。





