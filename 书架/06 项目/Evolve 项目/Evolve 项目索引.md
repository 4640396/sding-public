---
title: Evolve 项目索引
type: project-index
status: verified
created: 2026-07-17
updated: 2026-08-02
sensitivity: internal
project: Evolve 项目
project_id: evolve
sources:
  - .agents/skills/evolve/SKILL.md
  - "[[2026-07-17 Evolve 自维护可靠性优化与验收记录]]"
  - "[[Evolve 更名决策记录]]"
  - "[[2026-07-17 Evolve Skill 与项目更名验收记录]]"
  - "[[Project Evolve 独立项目归属决策记录]]"
  - "[[2026-07-17 Project Evolve 独立项目迁移与验收记录]]"
  - "[[2026-07-17 Evolve 知识落点约束与验收记录]]"
  - "[[2026-07-17 Graphify 外部缓存迁移与 Evolve 规则改造验收记录]]"
  - "[[2026-07-25 新设备 D 盘路径迁移与验收记录]]"
  - "[[2026-07-25 双机便携路径改造与验收记录]]"
  - "[[2026-07-25 知识库换机迁移与双机同步阶段总结]]"
  - "[[2026-08-02 Evolve 设计优先流程改造与验收记录]]"
  - "[[2026-08-02 Evolve 设计优先与 Product Design 编排总结]]"
---

# Evolve 项目索引

## 定位

Evolve 是跨项目复用的需求演进与授权编排工作流。它根据任务需要协调 GBrain、Graphify、原始资料核验、产品决策、代码实施、阶段验收和 Obsidian 沉淀，但不从属于 GBrain 或 Graphify。

实际 Skill 维护在 `.agents/skills/evolve`；本项目目录保存人类可读说明、规则决策、实施验收和历史复盘。旧名称 `Project Evolve` 保留在历史记录中供追溯。

## 稳定边界

- Obsidian Vault、源码、配置、官方资料和测试结果是事实依据。
- GBrain 负责知识检索与写入治理。
- Graphify 只在代码关系影响判断时作为辅助索引使用。
- Graphify 派生图谱统一保存在源码仓库与 Vault 外的项目专属操作系统缓存。
- Evolve 负责识别七步生命周期位置、授权边界和最小必要流程。
- 重要界面改造采用“真实页面审计 → 效果图比较与确认 → 结构化 Figma 开发源 → 从确认节点实施 → 视觉对照验收”的设计优先分支。

## 笔记导航

### 01 需求与决策

- [[Evolve 更名决策记录]]
- [[Project Evolve 独立项目归属决策记录]]
- [[2026-07-17 Project Evolve 规则完善决策记录]]

### 02 架构与实现

- [[Evolve 跨项目演进工作流 Skill]]

### 03 实施与验收

- [[2026-08-02 Evolve 设计优先流程改造与验收记录]]
- [[2026-07-25 双机便携路径改造与验收记录]]
- [[2026-07-25 新设备 D 盘路径迁移与验收记录]]
- [[2026-07-17 Graphify 外部缓存迁移与 Evolve 规则改造验收记录]]
- [[2026-07-17 Evolve 知识落点约束与验收记录]]
- [[2026-07-17 Evolve 自维护可靠性优化与验收记录]]
- [[2026-07-17 Evolve Skill 与项目更名验收记录]]
- [[2026-07-17 Project Evolve 独立项目迁移与验收记录]]
- [[2026-07-17 Project Evolve Skill 规则完善与验收记录]]

### 05 历史与复盘

- [[2026-08-02 Evolve 设计优先与 Product Design 编排总结]]
- [[2026-07-25 知识库换机迁移与双机同步阶段总结]]
- [[2026-07-17 Evolve Skill 优化总结]]
- [[2026-07-17 Project Evolve 独立项目拆分总结]]
- [[2026-07-16 Project Evolve 创建与使用复盘]]
- [[2026-07-16 Project Evolve 六步项目工作流对话总结]]
- [[2026-07-17 Project Evolve 七步流程确认与使用总结]]

没有内容的 `04 运维与迁移` 暂不创建。

## 使用入口

在 Codex 中调用 `$evolve`，然后用“分析、决策、实施、验收、总结并沉淀”等自然语言描述当前动作。旧 `$project-evolve` 入口不再保留。
