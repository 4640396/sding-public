---
title: git-ai 项目索引
type: index
status: verified
created: 2026-07-27
updated: 2026-07-27
sensitivity: internal
project: git-ai
project_id: git-ai
sources:
  - "[[git-ai 项目架构、业务流程与使用指南]]"
---

# git-ai 项目索引

## 项目定位

git-ai 是一个显式证据驱动的 Git AI 代码归因系统：Agent 在编辑时提交 checkpoint，后台 daemon 通过 Git trace2 异步识别提交和历史重写，最终将行级归因写入 `refs/notes/ai`。

## 架构与使用

- [[git-ai 项目架构、业务流程与使用指南]]：项目目标、系统架构、checkpoint 到 Git Note 的主流程、历史重写归因、安装使用、开发命令与未验证边界。

## 源码与派生索引

- 源码工作区：`$env:WORKS_ROOT\git-ai`
- 已核验源码版本：`e9e6bbd218c1f35405dedfd20e17b19fb4acb65e`
- Graphify 稳定项目标识：`git-ai`
- Graphify 图谱是 Vault 外可删除、可重建的派生缓存，不是本项目的事实来源。
