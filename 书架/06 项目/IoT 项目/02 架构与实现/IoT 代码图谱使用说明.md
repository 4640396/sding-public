---
title: IoT 代码图谱使用说明
type: tool-guide
status: verified
created: 2026-07-15
updated: 2026-07-25
sensitivity: internal
sources:
  - $env:WORKS_ROOT\iot 当前源码
  - Graphify 0.9.16 本地生成结果
  - 用户于 2026-07-17 要求将 Graphify 派生目录迁出源码项目
---

# IoT 代码图谱使用说明

## 定位

IoT 代码图谱是从当前源码生成的辅助索引，用于发现类、方法、调用关系、依赖和代码社区。它不是项目事实来源，任何结论都必须回到 `$env:WORKS_ROOT\iot` 当前源码和配置核验。

## 本机位置

```text
%LOCALAPPDATA%\Graphify\projects\iot\graphify-out
```

主要文件：

- `graph.html`：交互式关系图。
- `graph.json`：完整图数据。
- `GRAPH_REPORT.md`：自动生成的图谱摘要。

这些文件是派生数据，不复制进 Obsidian Vault，不进入稳定知识层，换电脑后从源码重新生成。

## 当前生成结果

- 生成时间：2026-07-15。
- 输入：143 个代码文件。
- 图谱：1883 个节点、4544 条关系、78 个社区。
- 模式：`--code-only`，Tree-sitter 本地解析，不调用云端模型，也不需要 Ollama。
- 已知缺口：一个 SQL 文件因未安装 `graphifyy[sql]` 可选依赖而未进入图谱。

## 查看与查询

在浏览器打开：

```text
%LOCALAPPDATA%\Graphify\projects\iot\graphify-out\graph.html
```

在 PowerShell 中先绑定 IoT 的外部图谱缓存，再查询：

```powershell
$env:GRAPHIFY_OUT = Join-Path $env:LOCALAPPDATA "Graphify\projects\iot\graphify-out"
graphify query "告警通知涉及哪些模块"
graphify explain "AlarmService"
graphify affected "AlarmService"
graphify path "AlarmService" "WeChatService"
```

不在项目目录时显式指定图文件：

```powershell
graphify query "告警通知涉及哪些模块" `
  --graph "$env:LOCALAPPDATA\Graphify\projects\iot\graphify-out\graph.json"
```

## 更新与重建

代码发生变化后：

```powershell
$env:GRAPHIFY_OUT = Join-Path $env:LOCALAPPDATA "Graphify\projects\iot\graphify-out"
graphify update "$env:WORKS_ROOT\iot"
graphify cluster-only "$env:WORKS_ROOT\iot" --graph "$env:GRAPHIFY_OUT\graph.json" --no-label
```

完整重新生成：

```powershell
$cacheRoot = Join-Path $env:LOCALAPPDATA "Graphify\projects\iot"
$env:GRAPHIFY_OUT = Join-Path $cacheRoot "graphify-out"
graphify extract "$env:WORKS_ROOT\iot" --code-only --out $cacheRoot
graphify cluster-only "$env:WORKS_ROOT\iot" --graph "$env:GRAPHIFY_OUT\graph.json" --no-label
```

如需解析 SQL：

```powershell
uv tool install "graphifyy[sql]" --reinstall
```

随后重新生成图谱。

## 使用规则

- `EXTRACTED` 表示从源码结构直接提取，仍需检查对应文件和上下文。
- `INFERRED` 只作为关联线索，不能单独支撑稳定结论。
- `AMBIGUOUS` 表示证据不唯一，回答时必须说明不确定性。
- `graph.json`、`GRAPH_REPORT.md` 和图谱可视化均不是事实来源。
- 稳定架构结论经源码核实后，再写入 IoT 项目对应模块笔记。
- 图谱只写入项目专属的操作系统缓存，源码仓库内不保存 `graphify-out/`。
- 仓库仍可保留 `graphify-out/` Git 忽略规则作为误生成兜底。

## 换机恢复

只迁移 `$env:WORKS_ROOT\iot` 源码和 Obsidian Vault，不需要备份操作系统缓存中的图谱。新电脑安装 `graphifyy` 后按外部缓存命令重新生成即可。
