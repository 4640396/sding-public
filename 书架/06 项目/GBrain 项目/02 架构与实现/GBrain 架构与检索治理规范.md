---
title: GBrain 架构与检索治理规范
type: governance
status: verified
created: 2026-07-15
updated: 2026-07-17
sensitivity: internal
sources:
  - AGENTS.md
  - 用户于 2026-07-15 确认的 GBrain + Graphify 分层方案
  - 用户于 2026-07-17 要求将 Graphify 派生目录迁出源码项目
---

# GBrain 架构与检索治理规范

## 目标

在不破坏 Obsidian Vault 唯一事实来源地位的前提下，使用 Graphify 提供关系发现能力，并由 GBrain 执行知识治理、混合检索、原文核验和答案组织。

## 分层架构

```text
Obsidian Vault（唯一事实来源）
        ↓
确定性筛选器
目录白名单 + status: verified + sensitivity: public|internal
        ↓
只读索引镜像 / 文件清单
        ↓
Graphify 建立辅助关系图
        ↓
GBrain 混合检索
图谱查询 + 原文搜索 + 直接读取来源
        ↓
答案 + 来源笔记 + 可信状态 + 不确定性
```

## 各层职责

### Obsidian Vault

- 保存原始 Markdown 和人工确认的稳定知识。
- 是结论、状态、来源和更新时间的唯一事实来源。
- Graphify 输出、索引镜像和 Agent 推断不得反向成为事实。

### 确定性筛选器

- 在 Graphify 之前执行，不允许 Graphify 自行决定索引范围。
- 读取目录、YAML Frontmatter 和敏感级别。
- 输出可复现的只读镜像及文件清单。

### Graphify

- 发现实体、笔记关系、关联路径和上下游线索。
- 仅作为可删除、可重建的辅助索引。
- 不负责判断一条内容是否为稳定事实。

### GBrain

- 执行本规范和知识库根目录 `AGENTS.md`。
- 组合图谱查询、精确全文搜索和原始笔记读取。
- 核验状态、更新时间、来源和冲突后组织答案。

## 索引准入规则

### 目录白名单

默认允许：

```text
书架/03 卡片盒
书架/04 知识地图
书架/05 领域
书架/06 项目
```

其中 `书架/06 项目` 只有符合 YAML 状态要求的稳定说明可以进入索引。

### YAML 要求

稳定索引只接受：

```yaml
status: verified
sensitivity: public # 或 internal
```

- `public` 可以按任务需要使用允许的处理后端。
- `internal` 只允许本地处理；Graphify 语义提取使用本地 Ollama。
- `confidential` 永不进入索引，也不得复制到外部服务。
- 缺少 `sensitivity` 或使用白名单之外敏感级别的笔记不进入稳定索引。
- `draft`、`reference`、`accepted`、`backlog` 和缺少 `status` 的笔记不进入稳定索引。

### 明确排除

```text
.obsidian/
书架/00 收集箱/
书架/01 工作台/
书架/07 资源库/02 素材库/
*.xlsx
sensitivity: confidential
```

排查记录、日记和收集材料可以在具体任务中作为证据或上下文直接读取，但不能因为被检索到就升级为稳定事实。

## Graphify 关系可信度

Graphify 中的边按下列方式使用：

- `EXTRACTED`：从原文或结构直接提取，仍需回到原始 Markdown 核实上下文。
- `INFERRED`：模型推断，只能作为检索线索，不能独立支撑事实结论。
- `AMBIGUOUS`：证据不唯一，必须明确标记不确定性并继续核验。

禁止仅凭 `graph.json`、`GRAPH_REPORT.md`、图谱可视化或模型生成说明形成稳定结论。

## 混合检索流程

```text
用户问题
 ├─ Graphify：发现相关实体、关联路径和上下游笔记
 ├─ 全文检索：查找准确名称、日期、错误码、字段、配置和原句
 └─ 原文读取：核实 YAML、更新时间、来源与完整上下文
```

执行顺序：

1. 判断问题涉及的项目、领域或概念。
2. 使用 Graphify 找到可能相关的节点和关系。
3. 使用精确搜索补充图谱容易遗漏的字符串事实。
4. 打开原始 Markdown，确认 `status`、`updated`、`sources` 和正文。
5. 检查不同笔记是否冲突。
6. 优先采用更新时间较新且有明确来源的稳定说明。
7. 输出答案、来源、可信度和不确定性。

当 Graphify 不可用或索引过期时，GBrain 必须退化为“全文搜索 + 原文读取”，不能停止基本检索，也不能假装图谱已经核验。

## 索引与事实隔离

- 索引镜像和 Graphify 输出必须位于 Vault 与源码仓库之外的操作系统缓存目录。
- 生成物不得进入 `书架/03 卡片盒`、`书架/05 领域`或稳定项目说明。
- `GRAPH_REPORT.md`、`graph.json` 和 `graph.html` 都是派生数据，可随时删除并重建。
- GBrain 不自动回写 Vault。新结论按照知识库写入规则进入收集箱、工作台或草稿笔记，人工确认后才能成为 `verified`。

当前默认运行目录：

```text
%LOCALAPPDATA%/GBrain/sding/index-input
%LOCALAPPDATA%/Graphify/projects/<project-id>/graphify-out
```

## 标准回答格式

```markdown
结论：……

可信度：高 / 中 / 低

依据：
- [[收车模块说明]]，status: verified，updated: 2026-07-10
- `来源笔记名称`，status: verified

图谱关系：
- 收车模块 → calls → 库存服务（EXTRACTED）

不确定性：
- 配置说明与较早排查记录存在冲突，以更新后的稳定说明为准。
```

没有图谱证据时省略“图谱关系”，没有实质不确定性时写“未发现明显冲突”，不得编造完整性保证。

## 可迁移性

- 人类可读规范保存在本笔记。
- Agent 工作流保存在 `.agents/skills/gbrain/SKILL.md`。
- 机器规则保存在 `.agents/skills/gbrain/references/policy.json`。
- 确定性筛选器保存在 `.agents/skills/gbrain/scripts/build_index.py`。
- 索引本身不迁移；换设备后从 Vault 重新生成。

本规范、Skill、策略文件和脚本修改时应同步检查，避免人类规则与机器行为不一致。
