---
title: 2026-07-16 Project Evolve 创建与使用复盘
type: conversation-record
status: draft
created: 2026-07-16
updated: 2026-07-16
sensitivity: internal
project: Evolve 项目
project_id: evolve
sources:
  - .agents/skills/project-evolve/SKILL.md
  - .agents/skills/project-evolve/agents/openai.yaml
  - "[[Evolve 跨项目演进工作流 Skill]]"
  - 2026-07-16 用户与 Codex 关于 GBrain、Graphify 和项目演进流程的对话
---

# 2026-07-16 Project Evolve 创建与使用复盘

## 本轮目标

解决跨项目开展需求改造时提示词过长、难以记忆，以及每次无条件运行 Graphify 可能增加等待时间的问题。最终形成可全局调用的 `$project-evolve` Skill，并明确从分析、知识沉淀到分阶段实施的最小操作方式。

## 已完成事项

### GBrain 全局可发现

GBrain 的维护副本仍位于 `C:\works\doc\sding\.agents\skills\gbrain`，全局入口位于 `C:\Users\46403\.codex\skills\gbrain`。全局入口通过目录联接指向维护副本，使 DCS、IoT、3x-ui-sub 等其他工作区能够发现同一份 Skill。

GBrain 的 Vault 定位规则已明确：从其他项目调用时仍使用 `C:\works\doc\sding` 作为规范 Vault，不把当前代码项目中的 `AGENTS.md` 误认为 Vault 根目录。

### Project Evolve Skill

已创建跨项目编排 Skill：

- 维护副本：`C:\works\doc\sding\.agents\skills\project-evolve`
- 全局入口：`C:\Users\46403\.codex\skills\project-evolve`
- 主规则：`.agents/skills/project-evolve/SKILL.md`
- 界面元数据：`.agents/skills/project-evolve/agents/openai.yaml`

本地验证包括 Skill 结构校验、从 `C:\works\3x-ui-sub` 读取全局入口，以及检查关键路由规则存在。当前规则见 [[Evolve 跨项目演进工作流 Skill]]。

## 最小使用方式

用户不需要记忆完整提示词，只需记住 `$project-evolve` 和当前动作：

```text
$project-evolve 分析 <项目名>：<需求>
$project-evolve 沉淀刚才的分析
$project-evolve 实施第一阶段
$project-evolve 总结并沉淀
$project-evolve 继续下一阶段
```

常用控制方式：

```text
$project-evolve --no-graphify <任务>
$project-evolve --reuse-graph <任务>
$project-evolve --refresh-graph <任务>
```

默认不要求用户提供参数：Skill 应根据任务类型选择最小必要流程。

## 标准项目演进循环

```text
提出需求
  → Analyze：查询背景并分析当前代码
  → Deposit：将分析、事实和待决策项分层沉淀
  → 用户确认范围及关键产品决策
  → Implement：实施一个明确阶段并运行测试
  → Deposit：记录实际改动、验证和遗留风险
  → 继续下一阶段
```

### 分析阶段

- GBrain 查询项目背景、历史记录和约束。
- 只有架构、调用链和影响范围确实需要时才调用 Graphify。
- 默认复用现有图谱；缺失、无法回答或明显过期时才刷新。
- 所有重要结论仍需返回原始 Markdown、源码或配置核验。
- “只分析”不授权修改代码。

### 沉淀阶段

- 用户明确说“沉淀、记录、同步”后才写入 Obsidian。
- 完整改造分析和待决策清单使用 `status: draft`。
- 当前代码事实只有重新核验原始源码并记录来源后，才可标记 `verified`。
- 规划、建议和目标架构不得描述成已经实现。
- 正式笔记进入对应项目目录；临时 `.gbrain-staging` 内容应先审核，再写入正式目录。
- 写入稳定项目索引时只增加必要导航，不擅自修改其他稳定结论。

### 实施阶段

- 每次只实施一个已经明确范围的阶段。
- 先读取已沉淀的分析、代码事实和待决策清单。
- 会影响实现的关键未决事项应在编码前明确。
- 完成后运行与改动风险匹配的测试，并区分已完成、未验证和下一步。
- 涉及真实支付、生产部署、删除数据等高风险动作时必须获得更明确的授权和约束。

## 3x-ui-sub 本轮示例

已形成用户中心与支付改造分析，内容分为：

- 完整改造分析：`draft`
- 待决策清单：`draft`
- 当前代码事实核验：在重新核验源码并记录来源后使用 `verified`

三份内容若仍位于 `.gbrain-staging`，下一步应先在 Codex 中审核差异，再要求 GBrain 将其正式写入 `书架/06 项目/3x-ui 项目`。正式写入和核验完成后，再清理本次 staging 文件。

第一阶段实施建议聚焦 Portal 骨架、用户认证和 PostgreSQL 迁移；暂不接真实支付，不执行生产部署。真实支付渠道、退款规则和计费方式仍属于需要用户明确确认的产品决策。

## 用户只需记住的规则

```text
分析 → 沉淀 → 实施第一阶段 → 总结并沉淀 → 继续下一阶段
```

如果连完整句子也不想记，可在 Codex 的 Skill 列表选择 `Project Evolve`，然后直接描述项目和需求。

## 治理说明

- 本记录是对本轮对话和当前实现状态的整理，尚未经过长期跨项目验证，因此保持 `status: draft`。
- [[GBrain 项目索引]] 和 [[GBrain 架构与检索治理规范]] 保持不变。
- 本轮未运行或刷新 Graphify，因为任务属于对话总结和 Obsidian 沉淀。
- 后续应以真实项目使用结果验证模式识别、图谱复用判断和沉淀路径是否稳定。

## 后续动作

1. 审核并正式写入 3x-ui-sub 的三份 staging 笔记。
2. 使用 `$project-evolve 实施 3x-ui-sub 第一阶段` 开始限定范围的开发。
3. 第一阶段完成后使用 `$project-evolve 总结并沉淀` 记录实际代码变更和测试结果。
4. 在第二个项目中复用该流程，观察是否需要调整 Skill 的触发示例和图谱刷新标准。
