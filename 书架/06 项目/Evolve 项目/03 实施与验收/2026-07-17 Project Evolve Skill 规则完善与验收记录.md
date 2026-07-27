---
title: 2026-07-17 Project Evolve Skill 规则完善与验收记录
type: implementation-record
status: verified
created: 2026-07-17
updated: 2026-07-17
sensitivity: internal
project: Evolve 项目
project_id: evolve
sources:
  - .agents/skills/evolve/SKILL.md
  - .agents/skills/evolve/agents/openai.yaml
  - .agents/skills/evolve/references/regression-cases.md
  - "[[2026-07-17 Project Evolve 规则完善决策记录]]"
  - 2026-07-17 本地 Skill 校验和内容回归输出
---

# 2026-07-17 Project Evolve Skill 规则完善与验收记录

## 结论

Project Evolve 已按用户确认的 14 项选择完成规则更新。Skill 结构通过官方校验，关键规则通过本地文本回归检查；人类可读说明和界面元数据已同步。本次未运行 Graphify，未修改其他项目源码，也未连接外部或生产服务。

## 实际修改

- 将主模型统一为七步生命周期，并把查询、总结、复盘和维护定义为辅助动作。
- 明确“继续下一阶段”不自动授权实施。
- 增加组合动作失败与沉淀处理、重复验收续验规则。
- 限定验收自动修复边界，设计或范围变化转入决策。
- 增加 Agent 受托代选的授权与高风险排除规则。
- 增加六类最低验证矩阵和三态阶段报告。
- 增加验证环境、操作、结果、未执行原因和源码版本要求。
- 统一 `sources`、`project` 与 `project_id` 的使用规则。
- 收紧 `verified` 的事实覆盖范围。
- 量化 Graphify 过期标准和 `--reuse-graph` 退化行为。
- 增加第三方官方资料核验要求。
- 新增 `.agents/skills/project-evolve/references/regression-cases.md`。
- 同步 `agents/openai.yaml` 和 [[Evolve 跨项目演进工作流 Skill]]。

## 验收证据

### 官方结构校验

最终运行 Skill Creator 的 `quick_validate.py`，结果：

```text
Skill is valid!
```

首次使用捆绑 Python 直接运行时缺少 `PyYAML`；改用 `uv run --with pyyaml` 后，Windows 默认文本编码导致 UTF-8 Skill 读取失败。设置独立工作区缓存和 `PYTHONUTF8=1` 后校验通过。这些属于校验运行环境问题，不是 Skill 结构缺陷。

### 内容回归

本地精确检查确认以下规则存在：

- 七步生命周期；
- “继续下一阶段”不构成实施授权；
- 验收小缺陷修复边界；
- 最低验证矩阵；
- 实现完成、验收通过、生产就绪三态；
- `sources`、`project_id` 和项目规范名称；
- 官方资料核验；
- `--reuse-graph` 退回源码；
- 回归用例入口；
- `agents/openai.yaml` 的默认提示显式包含 `$project-evolve`。

`SKILL.md` 共 151 行，低于 Skill Creator 建议的 500 行上限。

## 阶段状态

- 规则实现完成：是。
- 本地结构验收通过：是。
- 内容回归通过：是。
- 跨项目长期验证完成：否。
- 生产或外部系统影响：无。

## 未验证与后续观察

- 新规则尚未在第二个代码项目完成完整的分析、决策、实施和验收闭环。
- `project_id` 尚未追溯补齐到既有项目笔记，本次未自动整理无关历史记录。
- 最低验证矩阵仍需通过后续不同类型项目验证其粒度是否合适。
- “继续下一阶段”和验收修复边界需要通过真实短提示词继续观察稳定性。
