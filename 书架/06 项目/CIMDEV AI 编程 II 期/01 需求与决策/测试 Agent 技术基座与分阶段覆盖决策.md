---
title: 测试 Agent 技术基座与分阶段覆盖决策
type: decision
status: accepted
created: 2026-08-04
updated: 2026-08-04
sensitivity: internal
project: CIMDEV AI 编程 II 期
sources:
  - 用户于 2026-08-04 确认公司使用 CimiCode（OpenCode Fork 版本）并接入 GLM-5.1
  - 用户于 2026-08-04 确认首期优先覆盖 Java、Vue 等成熟技术栈，后续逐步寻找并验证 C++、VB 测试能力
  - "[[书架/06 项目/CIMDEV AI 编程 II 期/02 架构与实现/测试 Agent 技术方案]]"
---

# 测试 Agent 技术基座与分阶段覆盖决策

## 决策结论

测试 Agent 采用以下技术基座和实施顺序：

1. 以公司现有 **CimiCode CLI** 作为 Agent 运行入口。CimiCode 是 OpenCode 的 Fork 版本，已接入 GLM-5.1；
2. 独立 Test Agent 不重新建设大模型运行框架，而是由测试服务在服务器上调用 CimiCode CLI，编排知识查询、测试生成、真实执行和报告汇总；
3. 首期优先覆盖 **Java、Vue/TypeScript** 等测试生态成熟的技术栈，先完成一个系统的单元测试、回归测试、UI 测试和综合报告闭环；
4. 根据系统需要继续扩展 Python 等成熟技术栈；
5. **C++、VB 不从最终范围删除**，但不阻塞首期可用版本。后续分别通过真实系统 PoC 确认可用工具、执行环境、有效占比和不可自动化范围；
6. 只有 C++、VB 专项 PoC 通过并形成正式验收口径后，才将对应系统纳入规模接入和最终验收。

## 采用原因

- 公司已经具备 CimiCode + GLM-5.1 基座，优先复用能够避免重复建设 Agent 运行框架；
- Java、Vue/TypeScript 的构建工具、单元测试框架和 Web UI 自动化生态相对成熟，适合先验证端到端产品闭环；
- C++ 的编译器、链接库、硬件 SDK 和 Mock 条件因项目而异；
- VB 必须先区分 VB.NET 与 VB6，两者的构建环境和可自动化边界差异显著；
- 先形成可用产品，再用 PoC 处理遗留技术栈，可以降低首期范围失控风险，同时保留客户最终多技术栈目标。

## 完成口径

| 口径 | 定义 |
|---|---|
| 首期可用 | Java、Vue/TypeScript 系统可通过独立 Test Agent 完成单元、回归、UI 三类测试和报告闭环 |
| 专项验证完成 | C++、VB 分别在至少一个真实系统中完成环境、生成、执行、效果和限制验证 |
| 项目最终完成 | 16 个系统按照确认后的技术栈能力范围完成接入和验收 |

## 对实施的约束

- 不能把上游 OpenCode 的命令参数直接当作 CimiCode 事实；P0 必须核验公司实际 Fork 的非交互调用、退出码、流式日志、并发隔离和 Skill/Command 注册方式；
- GLM-5.1 负责理解、生成和失败分析，测试是否通过必须以 Maven、Vitest、Playwright、CMake、MSBuild 等真实工具结果为准；
- C++、VB 在 PoC 前不承诺达到 80% 有效用例占比，也不承诺固定规模接入工期；
- 首期可用不等同于 16 系统最终验收完成；对外汇报必须分别披露两个状态。

## 尚未决定

- 首个 Java + Vue/TypeScript 试点系统；
- CimiCode 实际版本及服务化调用契约；
- C++ 试点系统、编译器、测试框架和硬件依赖；
- VB 系统属于 VB.NET 还是 VB6；
- C++、VB 是否包含桌面 UI 自动化；
- 16 系统最终分批名单和各批次验收时间。

## 关联方案

- [[书架/06 项目/CIMDEV AI 编程 II 期/02 架构与实现/测试 Agent 技术方案]]
- [[书架/06 项目/CIMDEV AI 编程 II 期/02 架构与实现/cx-aicode 旧版本源码核验记录]]
- [[书架/06 项目/CIMDEV AI 编程 II 期/01 需求与决策/CIMDEV AI 编程 II 期系统知识库可执行方案]]

本决策状态为 `accepted`，表示实施方向已经确认，不表示相关能力已经开发、验收或生产就绪。
