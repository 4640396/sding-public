---
title: cx-aicode 旧版本源码核验记录
type: source-audit
status: verified
created: 2026-07-30
updated: 2026-07-30
sensitivity: internal
project: CIMDEV AI 编程 II 期
sources:
  - C:\works\cx-aicode\README.md
  - C:\works\cx-aicode\skills\process\cx-codereview\SKILL.md
  - C:\works\cx-aicode\commands\cx-codereview.md
  - C:\works\cx-aicode\agents\cx-code-reviewer.md
  - C:\works\cx-aicode\scripts\opencode\build-package.js
  - C:\works\cx-aicode\server\usage-api\README.md
  - C:\works\cx-aicode\tests\agents\test_agent_tools_allowlist.js
---

# cx-aicode 旧版本源码核验记录

## 1. 核验范围

2026-07-30只读检查 `C:\works\cx-aicode`。用户明确说明该目录不是最新版，因此本文只验证该版本的代码事实；用于确认流程形态和第四版业务逻辑，不证明最新版具有完全相同的文件、接口和约束。

仓库目录未包含 `.git`，无法记录commit或分支。使用Graphify对262个代码文件进行仓库外code-only提取，图谱位于系统缓存，不进入源码仓库或Vault；生成3364个节点、6436条边。重要结论均回到原始源码核验。

## 2. 已验证事实

### 2.1 产品形态

该版本cx-aicode是面向OpenCode的工程流程插件，包含：

- `/cx-*` commands；
- process、technical和business skills；
- 只读reviewer agents；
- OpenCode hooks和policy；
- Node.js状态、报告、Code Review和安装脚本；
- 独立Node.js usage-api；
- 自动测试和release打包流程。

因此它既不是单一Skill，也不存在已核验的Python运行后台。

### 2.2 Code Review真实链路

```text
/cx-codereview
→ skills/process/cx-codereview/SKILL.md
→ 确定scope与source files
→ resolve-reference.js生成Reference Inventory和reviewPasses
→ 主流程逐pass调用cx-code-reviewer
→ validate-agent-result.js校验pass result
→ finalize-report.js统一合并
→ write-report.js
→ update-status.js
→ spawn-upload.js可选上传使用统计
```

`finalize-report.js`及其下游负责结果收口，不适合作为Workspace知识查询入口。

### 2.3 Skill与Agent边界

process Skill属于编排层，负责参数、上下文、子Agent调用和原子脚本编排。`cx-code-reviewer`属于只读判断层，权限允许读取、检索和Skill，明确禁止编辑、Bash、任务委派、外部目录、Web访问和提问。仓库测试强制所有插件Agent遵守同一工具白名单。

因此Workspace查询应由执行process Skill的OpenCode主代理完成，再把显式、受控的知识上下文传给reviewer；不应放开reviewer网络权限。

### 2.4 usage-api边界

`server/usage-api`是Node.js服务，负责接收cx-aicode命令和Code Review使用事件，支持MySQL、PostgreSQL和Oracle。它不是Agent运行时、Workspace代理或知识库服务。其统计失败已有本地pending和降级机制。

### 2.5 打包影响

`scripts/opencode/build-package.js`会构建commands、skills、agents、hooks和scripts，并生成受管runtime manifest。任何Workspace改造都必须同时验证源码运行、`dist/opencode`产物、安装器和clean-install E2E，不能只修改源码Markdown。

## 3. 已确认接入判断

Code Review首期推荐在Reference Inventory与审查范围形成后、首次调用reviewer前查询Workspace一次：

```text
scope + spec + source files + reference inventory
→ 主流程构造一至三个业务知识问题
→ workspace_knowledge_search
→ knowledgeContext
→ 各reviewPass显式复用同一knowledgeContext
```

这样能够保持：

- reviewer只读权限不变；
- Reference Inventory仍是评分规则唯一权威范围；
- Workspace只提供业务事实、系统约束、版本和历史问题；
- 多pass不会重复查询和产生结果漂移；
- Workspace失败时可以继续原Code Review并披露降级；
- 现有报告、状态和门禁链路不被绕过。

## 4. 仍需复核

- 最新版cx-aicode的目录、契约、OpenCode版本和打包逻辑；
- OpenCode注册MCP或外部工具的正式配置方式；
- 项目到Workspace知识库和APP的配置落点；
- `knowledgeContext`是否进入reviewer输出契约、最终报告metadata或sidecar；
- Workspace真实认证、问答、流式返回、来源、版本和错误码；
- usage-api是否需要增加最小知识使用统计字段。

这些事项未在当前旧版本代码或Workspace材料中得到验证，不属于本文 `verified` 结论。

## 5. 对第四版方案的影响

第四版的技术边界成立：Workspace作为正式知识服务，cx-aicode保留原流程，OpenCode主代理查询知识，只读reviewer不直接联网。后续业务决策进一步明确为“系统范围分批、业务流程不分期”：先做一个项目和一套知识库，但必须覆盖需求到交付完整流程；Code Review接入点只是其中一个已由旧源码核验的具体触点。

需要修正的组件表述是：把“cx-aicode是单一Skill、Python后台负责记录”改为“cx-aicode是OpenCode工程流程插件、process Skill编排、主代理执行、reviewer只读判断、Node.js usage-api负责统计”。现行决策见 [[../01 需求与决策/CIMDEV AI 编程 II 期系统知识库可执行方案]]。
