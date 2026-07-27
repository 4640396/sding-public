---
title: GBrain 项目索引
type: project-index
status: verified
created: 2026-07-15
updated: 2026-07-17
sensitivity: internal
sources:
  - AGENTS.md
  - 用户于 2026-07-15 确认的 GBrain + Graphify 分层方案
---

# GBrain 项目索引

## 定位

GBrain 是本知识库的治理与检索层，不是事实存储。Obsidian Vault 是唯一事实来源；Graphify 是可删除、可重建的辅助关系索引。

## 架构

```text
Obsidian Vault（唯一事实来源）
        ↓ 确定性筛选
目录白名单 + status: verified + sensitivity: public|internal
        ↓
Vault 外只读索引镜像
        ↓
Graphify 辅助关系图
        ↓
GBrain 混合检索与原文核验
        ↓
答案 + 来源笔记 + 冲突 + 不确定性
```

## 稳定规则

完整规范见 [[GBrain 架构与检索治理规范]]。

- 默认索引 `书架/03 卡片盒`、`书架/04 知识地图`、`书架/05 领域`、`书架/06 项目`。
- 只有同时具有 `status: verified` 和 `sensitivity: public|internal` 的 Markdown 可以进入稳定索引；缺少任一字段时排除。
- `sensitivity: confidential` 永不进入索引；`internal` 仅限本地处理，Graphify 语义提取必须使用本地 Ollama。
- 收集箱、工作台、素材库、Excel 和 `.obsidian` 默认排除。
- Graphify 的推断边、报告和图文件不是事实，回答前必须核对原始 Markdown。
- 不自动回写 Vault；Agent 生成的新结论仍按知识库写入规则处理。
- 笔记冲突时优先采用更新时间较新且有明确来源的内容，并公开说明冲突。

## 可迁移实现

- 人类可读规范：[[GBrain 架构与检索治理规范]]
- 换机手册与问题复盘：[[GBrain 换机迁移与问题复盘]]
- 项目 Skill：`.agents/skills/gbrain/SKILL.md`
- 机器可读策略：`.agents/skills/gbrain/references/policy.json`
- 筛选与镜像脚本：`.agents/skills/gbrain/scripts/build_index.py`
- 换机与运维说明：`.agents/skills/gbrain/references/operations.md`
- Windows 引导脚本：`.agents/skills/gbrain/scripts/bootstrap_windows.ps1`
- 本机索引位于操作系统缓存目录，不随 Vault 同步；换机后依据本页和 Skill 重建。

## 笔记导航

### 02 架构与实现

- [[GBrain 架构与检索治理规范]]
- [[Evolve 项目索引]]：跨项目演进工作流，与 GBrain 为协作关系而非从属关系。

### 03 实施与验收

- [[2026-07-17 Vault 目录结构全量整改与验收记录]]
- [[2026-07-17 Vault 工作流收口与导航整改记录]]
- [[2026-07-17 Vault 目录治理规则与索引准入改造验收记录]]
- [[2026-07-17 Vault 导航去重与 Markdown 阅读样式统一验收记录]]

### 04 运维与迁移

- [[GBrain 换机迁移与问题复盘]]

### 05 历史与复盘

目录采用统一项目结构：`01 需求与决策`、`02 架构与实现`、`03 实施与验收`、`04 运维与迁移`、`05 历史与复盘`。项目索引保留在根目录；没有内容的分类不创建空目录。

## 使用

在 Codex 中调用 `$gbrain` 查询知识。重建索引前先预览：

```powershell
uv run .agents/skills/gbrain/scripts/build_index.py --vault . --dry-run
uv run .agents/skills/gbrain/scripts/build_index.py --vault .
uv run .agents/skills/gbrain/scripts/build_index.py --vault . --graphify --backend ollama
```
