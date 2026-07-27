---
title: Project Evolve 独立项目归属决策记录
type: decision-record
status: accepted
created: 2026-07-17
updated: 2026-07-17
sensitivity: internal
project: Evolve 项目
project_id: evolve
sources:
  - "[[Evolve 跨项目演进工作流 Skill]]"
  - "[[GBrain 项目索引]]"
  - 用户于 2026-07-17 确认将 Project Evolve 从 GBrain 项目拆分为独立项目并授权直接实施
---

# Project Evolve 独立项目归属决策记录

## 决策

将 Project Evolve 的人类可读知识、规则决策、实施验收和历史复盘从 `GBrain 项目` 迁移到独立的 `Project Evolve 项目`。

## 理由

- Project Evolve 已有独立生命周期、规则版本、回归用例和验收记录。
- 它跨多个代码项目复用，并协调 GBrain 与 Graphify，不是 GBrain 的子模块。
- 独立归属可以保持 GBrain“知识治理与检索层”的边界清晰。

## 边界

- 当时实际 Skill 保存在 `.agents/skills/project-evolve`；后续更名决策见 [[Evolve 更名决策记录]]。
- GBrain 项目只保留两者协作关系和指向独立项目索引的导航。
- 历史笔记保持原状态和标题，不因移动而提升事实状态。
