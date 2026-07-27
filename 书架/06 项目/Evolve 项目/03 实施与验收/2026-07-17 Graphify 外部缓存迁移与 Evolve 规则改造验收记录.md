---
title: 2026-07-17 Graphify 外部缓存迁移与 Evolve 规则改造验收记录
type: implementation-record
status: verified
created: 2026-07-17
updated: 2026-07-17
sensitivity: internal
project: Evolve 项目
project_id: evolve
sources:
  - AGENTS.md
  - .agents/skills/evolve/SKILL.md
  - .agents/skills/evolve/references/regression-cases.md
  - .agents/skills/evolve/agents/openai.yaml
  - C:\works\dcs\AGENTS.md
  - C:\works\iot\AGENTS.md
  - Graphify 0.9.16 CLI 帮助与 graphify/paths.py
  - "[[Evolve 跨项目演进工作流 Skill]]"
  - "[[IoT 代码图谱使用说明]]"
  - 用户于 2026-07-17 明确要求将 Graphify 派生目录迁出源码项目并实施、验收、总结和沉淀
---

# 2026-07-17 Graphify 外部缓存迁移与 Evolve 规则改造验收记录

## 结论

Graphify 代码图谱已从源码仓库和代码工作区父目录迁移到项目专属的操作系统缓存。迁移保留全部既有图谱，没有刷新或重建源码关系；源码工作区中的 `graphify-out` 已清零，Git 不再枚举其中的派生文件。

Evolve 已升级到 `2026-07-17.5`，后续默认使用 `%LOCALAPPDATA%/Graphify/projects/<project-id>/graphify-out`，禁止在源码仓库、代码工作区父目录或 Vault 中生成 Graphify 派生目录。

## 实际迁移

| 源位置 | 外部缓存标识 | 节点 | 边 |
|---|---|---:|---:|
| `C:\works\3x-ui-sub\graphify-out` | `3x-ui-sub` | 31 | 54 |
| `C:\works\dcs\graphify-out` | `dcs-legacy-mixed` | 116798 | 317292 |
| `C:\works\dcs\dcs-global-vehicle\graphify-out` | `dcs-global-vehicle` | 64304 | 187287 |
| `C:\works\dcs\dcs-global-vehicle-client\graphify-out` | `dcs-global-vehicle-client` | 5782 | 6919 |
| `C:\works\dcs\dcs-smmx-vehicle\graphify-out` | `dcs-smmx-vehicle` | 42437 | 117863 |
| `C:\works\dcs\dcs-smmx-vehicle-client\graphify-out` | `dcs-smmx-vehicle-client` | 4275 | 5223 |
| `C:\works\iot\graphify-out` | `iot` | 1883 | 4544 |

迁移前合计 7 个目录、13983 个文件，约 893.58 MB。迁移采用一对一移动，没有复制第二份图谱。

## 重复图谱处理

迁移前 `C:\works\dcs\graphify-out` 与 `dcs-global-vehicle` 仓库内图谱同时存在。父目录图谱包含后端目录，也包含根级 `src`、`package.json`、`static` 等混合来源，不能作为单仓库当前图谱。

处理结果：

- 单仓库当前图谱使用 `dcs-global-vehicle` 外部缓存。
- 混合旧图完整保留为 `dcs-legacy-mixed`，仅供追溯。
- Evolve 规则明确禁止混合图覆盖单仓库图谱。

## 规则改造

- Vault 根目录 `AGENTS.md` 增加 Graphify 外部缓存硬约束。
- Evolve Skill 升级到 `2026-07-17.5`，补充外部缓存定位、稳定项目标识、完整提取、查询、增量更新与旧图迁移规则。
- Evolve 界面默认提示与回归用例同步更新。
- [[Evolve 跨项目演进工作流 Skill]] 同步人类可读说明。
- [[IoT 代码图谱使用说明]] 改为外部缓存命令。
- GBrain 治理规范明确派生索引同时位于 Vault 与源码仓库之外。
- `C:\works\dcs\AGENTS.md` 和 `C:\works\iot\AGENTS.md` 更新当前图谱入口。
- DCS 四个 Git 仓库的 `.git/info/exclude` 增加 `graphify-out/` 本机兜底规则；IoT 继续使用既有 `.gitignore` 规则。

## 验收证据

2026-07-17 本地验收结果：

- 7 份外部 `graph.json` 均可解析，节点和边数量与迁移前一致。
- 7 份图谱的重复节点 ID、悬空边和自环均为 0。
- 每份外部缓存均记录对应 `.graphify_root`；混合旧图单独标记为 legacy。
- `C:\works` 递归检查 `graphify-out`：0 个。
- DCS 四仓库和 IoT 的 `git status -- graphify-out`：均为 0 条。
- DCS 四仓库的本机 exclude 与 IoT 的 `.gitignore` 均能命中 `graphify-out/`。
- 在 `C:\works\iot` 设置外部 `GRAPHIFY_OUT` 后执行 `graphify explain AlarmService` 成功，返回 `AlarmService.java` 及 56 条连接。
- Skill Creator 官方 `quick_validate.py`：`Skill is valid!`，退出码 0。
- 当前规则入口中，指向源码仓库内 `graphify-out` 的旧路径残留为 0。
- Vault 当前扫描 Markdown 128 份、Wiki 链接 439 条，缺失或同名歧义 0 条；本记录已进入 Evolve 项目索引。
- GBrain 确定性筛选 `--dry-run` 执行成功：纳入 62 份，按状态排除 38 份。

## Git 与现有改动边界

- 所有迁移目录均未被 Git 跟踪；迁移没有删除或修改项目源码。
- `.git/info/exclude` 是本机 Git 元数据，不产生工作树待提交记录。
- IoT 的 `AGENTS.md` 在本次任务开始前已经处于修改状态；本次只在既有 Graphify 章节中更新图谱路径与调用方式，没有覆盖其他未提交内容。
- `dcs-global-vehicle` 中既有 `.graphifyignore` 和其他源码改动不属于本次范围，保持原样。
- 全 Vault 样式扫描另发现 `书架/02 日记/未命名.md` 缺少 Frontmatter 和一级标题；该文件早于本次沉淀写入且不属于 Graphify 迁移范围，本次未修改。

## 未验证边界

- 本次验证的是目录迁移、图数据完整性、外部路径查询、Git 隔离和规则一致性。
- 没有执行完整 Graphify 重新提取或增量更新，避免在纯迁移任务中改变既有图谱内容。
- `dcs-legacy-mixed` 只验证文件完整和图结构健康，不承诺其语义范围适合作为当前项目图谱。
- 未运行生产部署、外部服务或业务功能测试；本次改造不涉及生产行为。

## 后续维护

- 每次 Graphify 调用先解析稳定项目标识，并设置项目专属的绝对 `GRAPHIFY_OUT`。
- 完整提取同时使用 `--out` 指向项目缓存父目录；查询和更新复用同一缓存。
- 收口时检查源码仓库及工作区父目录没有新建 `graphify-out`，并检查 Git 状态。
- 外部缓存无需迁移或备份；换机后从源码重新生成。
