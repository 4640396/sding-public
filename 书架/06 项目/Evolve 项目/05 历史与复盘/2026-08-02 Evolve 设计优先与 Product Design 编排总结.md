---
title: 2026-08-02 Evolve 设计优先与 Product Design 编排总结
type: retrospective
status: verified
created: 2026-08-02
updated: 2026-08-02
sensitivity: internal
project: Evolve 项目
project_id: evolve
sources:
  - .agents/skills/evolve/SKILL.md
  - .agents/skills/evolve/agents/openai.yaml
  - .agents/skills/evolve/references/regression-cases.md
  - "[[Evolve 跨项目演进工作流 Skill]]"
  - "[[2026-08-02 Evolve 设计优先流程改造与验收记录]]"
---

# 2026-08-02 Evolve 设计优先与 Product Design 编排总结

## 背景

本轮实践从登录后页面改造开始。有效做法不是先写代码再反复调样式，而是先审计真实页面、用效果图建立共同视觉目标、根据反馈融合方案，再把确认稿转为结构化 Figma 开发源并实施验收。该流程已从一次性实践升级为 Evolve `2026-08-02.3` 的稳定界面改造分支。

## 最终闭环

重要界面改造采用以下顺序：

```text
真实页面与流程证据
→ Product Design audit（用户要求审计、评估或反馈时）
→ Product Design ideate（三张独立效果图）
→ 用户选图、批注或要求融合
→ 确认唯一视觉目标
→ 结构化 Figma（Auto Layout、组件、变量、断点与状态）
→ Figma design-to-code
→ design-qa 对照 Figma 验收
→ Evolve 阶段报告与按需沉淀
```

Product Design 负责方向探索，Figma 负责可编辑设计源和开发上下文，Evolve 继续负责生命周期、授权边界、产品事实、安全约束、实施范围与知识落点。三者是编排关系，不互相替代。

## 核心规则

- 页面信息架构、关键路径、布局层级、视觉语言或多区域联动发生变化时，先设计后实施。
- 页面与素材可访问时直接开始审计和出图，不增加“是否开始设计”的空确认。
- 没有明确视觉目标时，`$product-design:ideate` 默认生成恰好三张独立方案，而不是把多个方向塞进一张图。
- 多方案必须由用户选择；“你直接改”“你看着办”或一般性代选委托不能授权 Agent 自动选稿。
- 用户喜欢多个方向的不同部分时，先生成统一融合稿并再次确认，不把冲突的视觉语言直接拼进代码。
- 已有明确且确认的效果图、截图或设计稿时，可以跳过三方案探索，直接建立结构化 Figma。
- Figma 交付不是把确认稿作为一张图片放入文件，而是使用 Auto Layout、复用组件或实例、变量、样式、必要断点和关键状态建立可编辑节点。
- 用户确认 Figma 节点后，以文件 URL、`fileKey` 和 `nodeId` 唯一标识开发设计源；开发通过 `$figma-design-to-code` 读取上下文。
- 效果图是设计决策证据，不代表功能已实现，也不得虚构后端能力、真实数据或削弱安全边界。
- `$design-qa` 是阻塞交付门禁；必须在相同视口和状态下比较确认稿与真实实现，并修复 P0、P1、P2 问题。
- Product Design 不可用或前置检查失败时必须明确报告；只有用户同意后才能降级，且不得冒充 Product Design 输出或已验证结果。

## 快速通道

以下任务通常不需要三方案探索：

- 明确的文案调整；
- 单个颜色、字号、间距等样式值修改；
- 范围清楚的小组件缺陷；
- 已经存在唯一确认稿的实现任务。

快速通道只跳过视觉探索，不跳过真实页面、交互状态和响应式验证。若局部改动实际影响关键路径、产品能力或安全边界，仍应返回决策阶段。

## 使用方式

常用提示可以保持自然语言：

- `$evolve 审计并重做登录后的用户中心`
- `$evolve 这三张效果图各有优点，请融合后给我看`
- `$evolve 把确认稿做成开发可用的 Figma，再实施并验收`
- `$evolve 把按钮文案和间距修一下`

Agent 应自动判断进入完整 Product Design 分支还是局部快速通道，用户无需记忆每个子 Skill 的调用顺序。

## 已验证与未验证边界

已验证：Evolve Skill 结构有效，Product Design 路由、默认三方案、用户选图门禁、结构化 Figma、确认节点、从 `get_design_context` 实施和 `design-qa` 门禁均已写入主规则、界面元数据、回归用例和人类可读说明。Figma 插件 `2.0.16` 已安装，Skill Creator 官方格式校验与独立前向测试均通过。

未验证边界：这些规则不代表未来每个业务项目的页面已经验收。实际任务仍须以该项目的真实页面、数据、权限、响应式环境和运行结果单独验证。
