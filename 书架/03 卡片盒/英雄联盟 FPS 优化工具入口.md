---
title: 英雄联盟 FPS 优化工具入口
type: tool
status: verified
created: 2026-07-25
updated: 2026-07-25
sensitivity: internal
sources:
  - "$env:WORKS_ROOT\\tools\\lol-fps-optimizer\\lol_fps_optimize.bat"
  - "[[../99 归档/旧 Codex 输出整理记录]]"
---

# 英雄联盟 FPS 优化工具入口

## 唯一入口

BAT 当前维护位置：

```powershell
& "$env:WORKS_ROOT\tools\lol-fps-optimizer\lol_fps_optimize.bat"
```

回滚命令：

```powershell
& "$env:WORKS_ROOT\tools\lol-fps-optimizer\lol_fps_optimize.bat" restore
```

回滚依赖的 `lol_fps_optimizer_backup` 与 BAT 保存在同一目录，不应单独移动或删除。

## 已核验事实

- 2026-07-25 从旧 Codex 日期化 `outputs` 目录迁移到 `$env:WORKS_ROOT\tools\lol-fps-optimizer`。
- 迁移后 BAT 的 SHA256 为 `9243A5EDEAB70F0D888C810425C020949D360DDFB271E0D927626188E25B82CB`。
- 旧位置的 BAT 和回滚备份目录已不存在，当前未保留第二个当前副本。

## 用途与风险

脚本会备份部分注册表和游戏配置，尝试启用 Windows 游戏模式、关闭后台录制、切换高性能电源方案、降低部分游戏画质，并清理超过指定天数的临时文件、崩溃转储和游戏日志。

> [!warning]
> 该工具包含删除和系统配置修改操作。重新运行前应保留同目录中的回滚备份，并人工审查脚本当前内容。`restore` 会固定切换到平衡电源方案，不会根据备份文件恢复迁移前的任意电源方案。

## 迁移验收边界

本次只验证文件与回滚备份已完整迁移、目标入口存在且旧入口消失；为避免再次触发清理和注册表修改，未在迁移验收中重新执行 BAT。
