---
title: 2026-07-17 Project Evolve 独立项目拆分总结
type: phase-summary
status: draft
created: 2026-07-17
updated: 2026-07-17
sensitivity: internal
project: Evolve 项目
project_id: evolve
sources:
  - "[[Project Evolve 独立项目归属决策记录]]"
  - "[[2026-07-17 Project Evolve 独立项目迁移与验收记录]]"
  - "[[Evolve 项目索引]]"
  - "[[GBrain 项目索引]]"
---

# 2026-07-17 Project Evolve 独立项目拆分总结

## 阶段结论

Project Evolve 已完成从 GBrain 项目子目录到独立项目的拆分。两者当前是协作关系：GBrain 负责 Vault 检索与写入治理，Project Evolve 负责跨项目生命周期、授权边界和最小必要流程编排。

## 实际完成

- 创建 `书架/06 项目/Project Evolve 项目` 和独立项目索引。
- 将 6 份既有专属笔记迁移到需求与决策、架构与实现、实施与验收、历史与复盘分类。
- 新增独立项目归属决策和迁移验收记录。
- 统一 9 份项目笔记的 `project`、`project_id` 和 `sources` 治理字段。
- 更新主页、项目总索引和 GBrain 项目索引。
- 删除 GBrain 项目中迁移后产生的空分类目录。
- 拆分当时保持 `.agents/skills/project-evolve` 维护路径和全局 Skill 目录联接不变；后续已按更名决策迁移到 `evolve`。

## 验收证据

- 本次范围内 12 份笔记的 Wiki 链接均可解析。
- Project Evolve 项目 9 份笔记的项目身份字段检查通过。
- 指向旧 GBrain 项目下 Project Evolve 笔记的显式路径为零。
- Skill Creator 官方校验结果为 `Skill is valid!`。
- 拆分验收当时全局 `$project-evolve` 入口仍指向 Vault 内维护副本；后续入口改为 `$evolve`。

## 未验证项

- 未刷新 GBrain 本地索引；需要时应单独授权并执行确定性筛选流程。
- 未在第二个代码项目完成新版规则的完整分析、决策、实施和验收闭环。
- 全 Vault 严格扫描识别到两个既有特殊引用：一个主页相对 Wiki 路径和一个图片附件；它们不属于本次迁移，也不是本次产生的问题。

## 遗留风险

- 若外部工具保存了旧的绝对笔记路径，Wiki 链接检查无法覆盖这些工具的私有配置。
- 历史复盘仍保留 `draft`，移动不改变其事实状态。
- GBrain 索引未刷新前，缓存中的旧目录位置可能暂时滞后，但 Vault 原文已经完成迁移。

## 下一步

当前拆分阶段已经收口。后续可在实际跨项目任务中验证新版 Project Evolve 规则；只有检索新位置确实需要时，再单独刷新 GBrain 本地索引。
