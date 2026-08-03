---
title: CIMDEV AI 编程 II 期项目索引
type: project-index
status: draft
created: 2026-07-29
updated: 2026-08-04
sensitivity: internal
sources:
  - 用户提供的 CIMDEV AI 编程 II 期项目规划书
  - 用户提供的 Workspace 产品、开放平台、Agent、RAG 工作流与知识问答格式材料
---

# CIMDEV AI 编程 II 期项目索引

## 项目定位

本项目拟基于现有私有化 Workspace，为 CIMDEV 与 YMS 的 16 个核心系统建设可供 AI 使用、可追溯、带版本并可持续维护的系统知识资产，并与 cx-aicode、后续测试 Agent 集成。

当前材料仍处于方案确认和实施准备阶段。第四版系统知识库方案已于2026-07-30确认业务逻辑和实施顺序，状态为 `accepted`；项目索引、代码实施设计和业务验证方案仍为 `draft`，Workspace真实API和代码实施尚未验收。

## 当前方案

- [[03 实施与验收/汇报材料/CIMDEV AI 编程 II 期系统知识库业务沟通材料 v4.pptx|系统知识库业务沟通材料 v4]]：用于业务沟通，说明客户诉求、Workspace 与 cx-aicode 的关系、实现难点、完整接入流程和近期行动项
- [[01 需求与决策/系统知识库需求设计实现实施说明|系统知识库需求设计实现实施说明]]：按需求、设计、实现和实施顺序解释第四版，适合作为首次阅读和近期行动手册
- [[01 需求与决策/CIMDEV AI 编程 II 期系统知识库可执行方案|系统知识库可执行方案 v4]]：Workspace 可行性、知识建设、持续更新、Skill-Agent-Workspace 集成、排期、职责与验收
- [[01 需求与决策/系统知识库首期业务验证实施方案|单项目完整流程验证实施方案]]：一个项目、一套知识库、需求到交付全流程对照验证和推广前证据要求
- [[02 架构与实现/cx-aicode 与 Workspace 代码实施方案|cx-aicode 与 Workspace 代码实施方案]]：具体开发工作包、工具契约、Skill改造、测试、Sprint和开工输入
- [[02 架构与实现/cx-aicode 旧版本源码核验记录|cx-aicode 旧版本源码核验记录]]：OpenCode插件形态、Code Review真实链路、Agent权限、usage-api边界和Workspace推荐接入点
- [[01 需求与决策/测试 Agent 技术基座与分阶段覆盖决策|测试 Agent 技术基座与分阶段覆盖决策]]：采用 CimiCode + GLM-5.1，首期 Java/Vue，C++/VB 后续专项 PoC
- [[02 架构与实现/测试 Agent 技术方案|测试 Agent 技术方案]]：独立 Test Agent、三类测试流水线、知识联动、实施周期、验证标准与风险
- [[02 架构与实现/测试 Agent 与 CimiCode 对接方案|测试 Agent 与 CimiCode 对接方案]]：本机 CLI 适配、组件职责、结构化事件契约、Workspace 上下文和部署演进边界
- [[03 实施与验收/测试 Agent PC 初版实施记录|测试 Agent PC 初版实施记录]]：Electron + Vue 3 桌面原型、CimiCode CLI 隔离适配、验证证据与未实现边界

## 版本历史

- [[05 历史与复盘/系统知识库可执行方案 v1|系统知识库可执行方案 v1]]：首次完整知识工程与治理蓝图，已被替代
- [[05 历史与复盘/系统知识库可执行方案 v2|系统知识库可执行方案 v2]]：修正 cx-aicode 与 Workspace 职责边界，已被替代
- [[05 历史与复盘/系统知识库可执行方案 v3|系统知识库可执行方案 v3]]：采用单系统、单APP、单阶段验证，但错误假设Python后台承担知识适配，已被替代
- [[05 历史与复盘/方案版本演进记录|方案版本演进记录]]：四版差异、修订原因与后续版本管理规则

## 当前状态

- 已基于客户提供材料完成方案级可行性分析。
- Workspace 已有资料能够支持内容读写、知识库上传、RAG 问答、APP 运行、Agent、Skill 与 Sandbox 等基础能力。
- 知识替换与下线、原始检索片段及引用返回、发布触发和正式审批仍需在客户实际环境中完成 PoC。
- cx-aicode旧版本源码已核验：它是OpenCode工程流程插件，process Skill负责编排，主代理执行工具调用，只读reviewer负责判断，Node.js usage-api负责统计。
- 建设范围按系统分批，业务流程不分期：先完成一个项目、一个知识库、一个通用问答APP、一个统一知识工具和cx-aicode从需求到交付完整流程；知识维护Agent延后到完整流程价值验证之后。
- 未开始实施，未完成任何系统知识录入或 cx-aicode 改造。
- 测试 Agent 已完成 PC 操作台初版代码及本地构建、测试、安全审计和窗口启动复验；真实 CimiCode、Workspace 与 Java/Vue 测试链路尚未接入，不能视为测试能力 PoC 已通过。

## 下一步

先验证一个项目的Workspace知识问答，再将同一知识工具完整接入需求分析、方案设计、任务拆解与实现、Code Review、测试验证和交付收尾，开展阶段与端到端有无知识对照；完整流程通过后建设知识维护闭环，最后进入3个代表系统和16个系统分批推广。实施前需用最新版cx-aicode和Workspace真实API复核接口细节。
