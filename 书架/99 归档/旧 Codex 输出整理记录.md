---
title: 旧 Codex 输出整理记录
type: archive
status: verified
created: 2026-07-25
updated: 2026-07-25
sensitivity: internal
sources:
  - "C:\\Users\\ma302\\Documents\\Codex"
  - "$env:WORKS_ROOT\\tools\\lol-fps-optimizer\\lol_fps_optimize.bat"
---

# 旧 Codex 输出整理记录

## 核验范围

2026-07-25 对 `C:\Users\ma302\Documents\Codex` 做只读递归盘点。15 个日期目录中共有 11 个文件，实际内容分为 B 端权限管理系统和英雄联盟 FPS 优化两组。

## 已沉淀

- B 端权限管理系统的完整 PRD 已写入 [[../06 项目/B端权限管理系统/B端权限管理系统项目索引|B端权限管理系统项目]]，状态保持为 `draft`。
- 两份 HTML 原型已登记来源与 SHA256，未作为事实源复制进 Vault。

## 未复制进 Vault

- `lol_fps_optimize.bat`：可执行源码工件，不复制进 Vault；当前唯一入口为 `$env:WORKS_ROOT\tools\lol-fps-optimizer\lol_fps_optimize.bat`。
- 回滚所需的注册表导出、游戏配置备份、电源方案和运行日志已与 BAT 一起迁移到同一工具目录，不进入 Vault。`dxdiag.txt` 仍作为原始诊断证据保留在旧 Codex 输出中。

## FPS 脚本审查摘要

该脚本会备份部分用户注册表与游戏配置，开启 Windows 游戏模式、关闭后台录制、切换高性能电源方案、修改游戏画质配置，并递归删除一定时间以前的临时文件、崩溃转储和游戏日志。脚本提供 `restore` 分支，但恢复时固定切换到平衡电源方案，并不读取已备份的原电源方案。

> [!warning]
> 脚本包含删除临时文件和修改注册表、游戏配置的操作。再次运行前应先人工审查；本次整理未执行该脚本，也未验证其优化效果或恢复完整性。

## 原目录处置

首次整理时没有删除或移动源目录。2026-07-25 后续整理已将 FPS BAT 及其回滚备份目录移至 `$env:WORKS_ROOT\tools\lol-fps-optimizer`，旧 `outputs` 中不再保留当前脚本副本。HTML 原型和其他历史证据不在本次迁移范围。

当前工具入口和使用边界见 [[../03 卡片盒/英雄联盟 FPS 优化工具入口]]。
