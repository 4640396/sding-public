---
title: GBrain 换机迁移与问题复盘
type: runbook
status: verified
created: 2026-07-15
updated: 2026-07-25
sensitivity: internal
sources:
  - AGENTS.md
  - GBrain 架构与检索治理规范
  - Graphify-Labs/graphify 官方 README（2026-07-15 核对）
  - 2026-07-15 本机安装与验证记录
  - "[[2026-07-25 双机便携路径改造与验收记录]]"
---

# GBrain 换机迁移与问题复盘

## 一句话迁移方案

完整复制 Obsidian Vault，在新电脑安装 Obsidian、Codex、uv 和 `graphifyy`，然后运行随 Vault 携带的引导脚本。GBrain Skill、规则、筛选器和迁移手册随 Vault 迁移；本机索引不迁移，重新生成。

当前 Vault：

```text
$env:WORKS_ROOT\doc\sding
```

## 必须复制的内容

复制整个 Vault，并确保隐藏目录没有遗漏：

```text
sding/
├─ .obsidian/                 Obsidian 配置
├─ .agents/skills/gbrain/     GBrain Skill、规则和脚本
├─ AGENTS.md                  知识库总规则
├─ 主页.md
└─ 书架/                      原始知识
```

不要只复制 `书架`。缺少 `.agents` 会丢失 `$gbrain`，缺少 `.obsidian` 会丢失 Obsidian 工作区配置。

## 不需要复制的内容

以下内容是可重建的本机派生数据：

```text
%LOCALAPPDATA%\GBrain\sding\index-input
%USERPROFILE%\.codex\skills\graphify
%USERPROFILE%\.local\bin\graphify.exe
```

不要把 API Key、密码、Cookie、令牌或私钥写进 Vault 后迁移。

## 新电脑恢复

1. 安装并登录 Obsidian 与 Codex。
2. 把整个 `sding` 复制到新电脑的目标位置。
3. 在 Obsidian 中选择该目录作为 Vault。
4. 在 PowerShell 进入 Vault 根目录。
5. 运行：

```powershell
powershell -ExecutionPolicy Bypass -File .agents\skills\gbrain\scripts\bootstrap_windows.ps1
```

6. 重启 PowerShell 和 Codex，在新任务中调用 `$gbrain` 或 `$evolve`。

引导脚本会优先使用有效的 `WORKS_ROOT`；变量缺失时，从当前 Vault 的 `<WORKS_ROOT>\doc\sding` 结构自动推导并保存到用户环境。它还会安装或确认 `uv`、安装 `graphifyy`、注册 GBrain、Evolve 与 Graphify 到 Codex，并重建经过治理规则筛选的镜像。它不会安装 Ollama，也不会自动进行语义建图。

## Graphify、GBrain 与 Ollama 的结论

### Graphify 不强制要求 Ollama

Graphify 官方的基础安装是：

```powershell
uv tool install graphifyy
graphify install --platform codex
```

Ollama 只是可选的本地模型后端，不是 Graphify CLI 的依赖。

### “30 秒”指快速安装与启动

官方“Get started (30 seconds)”描述的是安装 CLI、注册 Skill 和发起建图。实际完成时间取决于文件数量、内容类型、模型速度和机器性能，不能理解为任意知识库都在 30 秒内完成。

### 代码与 Markdown 的处理不同

- 程序代码由 Tree-sitter AST 在本地解析，不需要 LLM，也不需要 Ollama。
- Markdown、PDF、图片和视频的语义关系需要当前 AI 助手的模型或显式配置的模型后端。
- Obsidian Vault 主要是 Markdown，因此“语义图谱是否走云端或本地”是治理选择，不是安装要求。

### 为什么当前脚本仍限制 Ollama

当前 `build_index.py --graphify` 是无头批处理入口。为了避免 `sensitivity: internal` 在未确认的情况下被发送到云端，它只允许本地 `ollama` 后端。这是 GBrain 的保守安全策略，不是 Graphify 官方要求。

没有 Ollama时：

- `$gbrain` 仍可使用全文搜索和原文读取；
- 可以构建安全镜像和清单；
- 只是暂不执行 internal Markdown 的无头语义建图。

如果未来决定允许当前 Codex 模型处理 `internal`，必须同时修改治理规范、`policy.json`、Skill 和筛选脚本，不能只绕过脚本参数。

## 当前稳定架构

```text
Obsidian Vault（唯一事实来源）
        ↓
确定性筛选器
        ↓
Vault 外只读镜像 + 清单
        ↓
Graphify（可选辅助图谱）
        ↓
GBrain：图谱 + 全文搜索 + 原文核验
        ↓
答案 + 来源 + 可信度 + 不确定性
```

## 换机后验证清单

```powershell
uv --version
graphify --version
Test-Path .agents\skills\gbrain\SKILL.md
uv run .agents\skills\gbrain\scripts\build_index.py --vault . --dry-run
```

预期结果：

- 能显示 uv 和 Graphify 版本；
- GBrain Skill 文件存在；
- 预检只纳入 `status: verified` 且非 `confidential` 的白名单 Markdown；
- Codex 重启后可以调用 `$gbrain`。

## 常见问题

### `graphify` 找不到

```powershell
uv tool update-shell
```

关闭并重新打开 PowerShell。

### `$gbrain` 不出现

确认从 Vault 根目录打开 Codex，并检查：

```text
.agents/skills/gbrain/SKILL.md
```

然后重启 Codex。

### 没有 `graph.json`

这不代表 GBrain 失效，只表示 Graphify 辅助图尚未生成。GBrain 应自动退化为全文检索和原文读取。

### 换机后路径不同

脚本通过当前 Vault 根目录推导 `WORKS_ROOT`，不依赖旧电脑的 `C:\works\doc\sding`。当前导航和执行命令统一使用 `$env:WORKS_ROOT`；历史证据仍保留当时的绝对路径。若显式配置的变量指向不存在的目录，脚本会停止并明确报错。

## 权威文件

- 总规则：`AGENTS.md`
- 人类治理规范：[[GBrain 架构与检索治理规范]]
- 项目入口：[[GBrain 项目索引]]
- Agent 工作流：`.agents/skills/gbrain/SKILL.md`
- 机器策略：`.agents/skills/gbrain/references/policy.json`
- 筛选脚本：`.agents/skills/gbrain/scripts/build_index.py`
- 换机脚本：`.agents/skills/gbrain/scripts/bootstrap_windows.ps1`

修改安全策略时必须同步检查上述文件。
