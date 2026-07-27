---
title: 2026-07-17 Evolve Skill 与项目更名验收记录
type: implementation-record
status: verified
created: 2026-07-17
updated: 2026-07-17
sensitivity: internal
project: Evolve 项目
project_id: evolve
sources:
  - "[[Evolve 更名决策记录]]"
  - "[[Evolve 项目索引]]"
  - .agents/skills/evolve/SKILL.md
  - .agents/skills/evolve/agents/openai.yaml
  - .agents/skills/evolve/references/regression-cases.md
  - 2026-07-17 本地 Skill 校验、目录联接、路径和 Wiki 链接检查结果
---

# 2026-07-17 Evolve Skill 与项目更名验收记录

## 结论

Skill 和知识库项目已从 `project-evolve` / “Project Evolve”更名为 `evolve` / “Evolve”。旧全局 Skill 入口已移除，不提供兼容别名；历史正文中的旧名称按时间语境保留。

## 实际修改

- Skill 维护目录改为 `.agents/skills/evolve`。
- Skill frontmatter 改为 `name: evolve`。
- 界面名称和默认提示改为 `Evolve` 与 `$evolve`。
- 回归用例和当前短提示词改为 `$evolve`。
- 全局目录联接改为 `C:\Users\46403\.codex\skills\evolve`。
- 知识库项目目录改为 `书架/06 项目/Evolve 项目`。
- 项目索引和当前 Skill 说明分别改为 `Evolve 项目索引.md`、`Evolve 跨项目演进工作流 Skill.md`。
- 项目笔记统一使用 `project: Evolve 项目` 和 `project_id: evolve`。
- 主页、项目总索引和 GBrain 协作入口同步更新。

## 验收证据

- Skill Creator 官方 `quick_validate.py`：通过。
- 新 Skill 目录存在，旧 Skill 目录不存在。
- 新全局入口为指向 Vault 维护副本的目录联接。
- 旧全局 `$project-evolve` 入口不存在。
- 当前 Skill、界面元数据和回归用例不存在旧调用标识。
- Evolve 项目笔记 Frontmatter 身份检查通过。
- 本次范围内 Wiki 链接和导航检查通过。
- 指向旧项目目录的当前显式路径为零；历史验收记录中的旧路径作为当时事实保留。

## 阶段状态

- 更名实现完成：是。
- 本地验收通过：是。
- 新调用入口可发现：是。
- GBrain 索引刷新：未执行。
- 生产或外部系统影响：无。

## 未验证与风险

- 当前对话中附带的旧 Skill 路径在更名后失效，后续应从 Skill 列表选择 `$evolve`。
- 外部工具若私下保存旧绝对路径，需要由对应工具自行更新。
- GBrain 缓存索引在刷新前可能保留旧路径，但 Vault 原文和当前导航已完成更名。
