---
title: Evolve 跨项目演进工作流 Skill
type: implementation-note
status: verified
created: 2026-07-16
updated: 2026-07-25
sensitivity: internal
project: Evolve 项目
project_id: evolve
sources:
  - .agents/skills/evolve/SKILL.md
  - .agents/skills/evolve/agents/openai.yaml
  - .agents/skills/evolve/references/regression-cases.md
  - "[[2026-07-17 Evolve 自维护可靠性优化与验收记录]]"
  - "[[2026-07-17 Project Evolve 规则完善决策记录]]"
  - "[[2026-07-17 Project Evolve Skill 规则完善与验收记录]]"
  - "[[2026-07-17 Evolve 知识落点约束与验收记录]]"
  - "[[2026-07-17 Graphify 外部缓存迁移与 Evolve 规则改造验收记录]]"
---

# Evolve 跨项目演进工作流 Skill

## 定位

`$evolve` 是跨项目复用的轻量编排 Skill。它根据用户本次授权选择最小必要流程，协调 GBrain、Graphify、原始资料核验、产品决策、代码实施、阶段验收和 Obsidian 沉淀。

- GBrain 负责 Vault 检索、来源核验和写入治理。
- Graphify 只在代码关系确实影响判断时辅助发现架构、依赖、调用链和影响范围。
- Evolve 负责识别生命周期位置、授权边界和下一步动作，不改变事实来源与安全规则。

当前规则版本为 `2026-07-17.5`，主规则位于 `.agents/skills/evolve/SKILL.md`。

## 七步生命周期

```text
收集 → 分析 → 决策 → 实施 → 验收 → 沉淀 → 归档
```

七步表示完整项目生命周期，不要求每个请求机械执行全部步骤。查询、总结、复盘和维护是辅助动作。

关键授权边界：

- “只分析”不授权修改代码或写入 Vault。
- “继续下一阶段”只检查阶段状态并提出范围，不自动授权实施。
- “总结”只有同时要求记录、同步或沉淀时才写入。
- 验收可修复当前阶段内、不改变 accepted 设计的小缺陷；若涉及产品行为、数据模型、公共接口、安全边界或阶段范围变化，必须转入决策。
- 用户明确要求“先分析，可以优化就直接实施”时，可在不改变 accepted 产品边界的前提下连续执行分析、实施、验收和沉淀。

## 决策与状态

- 用户逐项确认，或明确委托 Agent 在限定范围内代选后，相关选择可记录为 `accepted`。
- Agent 代选必须记录授权依据、范围、理由和可逆性；费用、法律、真实支付、生产数据和重大安全降级不得通过一般性委托代选。
- `accepted` 表示已决定，不代表已实施。
- `verified` 只保证笔记中明确标出的原始事实和验证结果，不自动覆盖建议、计划或未验证边界。
- 未确认但完整、需要长期追溯的 `draft` 可进入项目目录；零散、正在编辑或等待确认的材料进入 `书架/01 工作台/<项目名>/`。

## 实施与验收

实施只覆盖用户授权的明确阶段。验收根据改动类型选择最低验证，覆盖数据库迁移、认证与会话、外部适配器、支付、配置部署或跨模块回归中的相关项目。

每次阶段报告必须区分：

- 实现完成；
- 验收通过；
- 生产就绪；
- 未验证项和剩余风险。

验证记录包含环境、命令或操作、结果、未执行原因和源码版本。优先使用 Git commit；没有可用 Git 历史时使用关键文件哈希。

## Graphify 与外部资料

Graphify 默认复用，不主动刷新。目标模块缺失、源码晚于图谱、关键路径缺失、图谱与源码冲突，或用户显式要求刷新时，才将图谱视为缺失或过期。`--reuse-graph` 模式下图谱不足时退回全文搜索和源码读取，不擅自刷新。

代码图谱是可删除、可重建的派生缓存，不属于源码或知识工件。默认位置为 `%LOCALAPPDATA%/Graphify/projects/<project-id>/graphify-out`；完整提取、增量更新和查询均通过绝对 `GRAPHIFY_OUT` 或显式图文件路径复用该缓存。源码仓库、代码工作区父目录和 Vault 均不得生成 `graphify-out`。旧图迁移前必须核对来源和重复关系，混合旧图不得覆盖单仓库图谱。

涉及支付、第三方 API、法规、标准、产品版本或商户能力时优先核验官方资料和目标环境。没有官方证据时保留为未验证项，真实接入前重新核验。

## Obsidian 沉淀

- Frontmatter 来源字段统一使用 `sources`。
- `project` 使用项目索引中的规范显示名；有稳定机器标识时同时使用 `project_id`。
- 沉淀结束检查分类、必要的项目索引导航、Wiki 与 Markdown 链接、替代关系、Frontmatter 和状态。
- 只有确需导航时才修改项目索引，不自动整理无关项目。
- 任何项目都必须先区分源码工件与知识工件，并从规范 Vault 的项目索引定位目标。DCS、IoT、3x-ui、GBrain、Evolve 及以后新增项目的源码工作区都不是知识沉淀位置，不得在其中另建 Markdown、独立 SQL 或“交付目录”知识副本。
- 需求分析、决策、执行手册、排查记录、验收证据和仅供人工执行的 SQL 属于知识工件；Vault 内 SQL 使用 Markdown `sql` 代码块。
- 应用直接消费的源码、配置和数据库迁移脚本仍留在代码仓库；Vault 只记录来源路径和验证结果。
- Graphify、GBrain 等派生索引统一保存在操作系统缓存目录，不作为源码或知识副本进入仓库与 Vault。
- 收口时反查源码工作区与 Vault，确保没有并行知识副本、当前引用不指向临时目录，并为长期项目工件补充必要入口。

## 维护与验证

修改 Skill 时同步检查：

- `.agents/skills/evolve/SKILL.md`
- `.agents/skills/evolve/agents/openai.yaml`
- `.agents/skills/evolve/references/regression-cases.md`
- 本说明和 Evolve 项目索引

运行 Skill Creator 官方 `quick_validate.py`，并用回归用例检查模式、授权、Graphify 和沉淀行为。变更历史与验收证据保存在项目实施验收记录中，不在 Skill 目录维护额外日志。

### 自维护可靠性

- 修改 Evolve 自身时使用 Skill Creator，并检查 Skill 文件夹名、frontmatter `name`、界面默认提示和 `$调用名` 一致。
- 重命名或迁移前盘点源路径、目标路径、反向引用、全局入口和权限；受保护目录需要批准时先申请。
- 多位置迁移不假设为原子操作；部分失败后重新读取实际状态，再决定后续步骤。
- 历史记录中的旧名称属于证据，不等同于当前残留。验收关注旧物理入口、当前调用示例和当前导航。
- 链接验收先检查本次范围；全 Vault 既有问题单独报告，不自动归因或顺带修复。
- 没有可用 Git 仓库时记录准确文件清单，必要时使用关键文件哈希。
- 先取得验证结果，再创建 `verified` 验收记录；提前记录过程时使用 `draft`。

## 本机实现

- 维护副本：`$env:WORKS_ROOT\doc\sding\.agents\skills\evolve`
- 全局入口：`%USERPROFILE%\.codex\skills\evolve`
- 全局入口通过目录联接指向 Vault 内维护副本，不存在第二份需要人工同步的 Skill 内容。
