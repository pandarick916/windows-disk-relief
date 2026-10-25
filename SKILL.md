---
name: windows-disk-relief
description: 把 Windows C 盘上的大文件夹（AI agent 家目录 .codex/.workbuddy/.claude、模型与缓存 .cache/.ollama、node_modules、Docker/WSL 等）安全搬到其他盘，并在原位建立目录 junction（传送门），让程序无感继续使用。含只读扫描、迁移、建链、校验、回滚。当用户提到「C 盘满了/红了、清 C 盘、给文件夹搬家、junction、mklink、目录链接、模型缓存太大、.codex/.workbuddy 占空间」时使用。
agent_created: true
---

# Windows 磁盘减负（junction 搬家）

## Overview

C 盘默认是「家目录 + AppData + 程序安装」的落脚点。AI 工具（尤其 agent）的家目录、缓存、模型、日志会持续吃 C 盘，**改工作目录也挡不住**——因为工作目录管「项目放哪」，用户目录管「agent 自己住哪」。

本技能：把大文件夹**搬到别的盘**，在原位建一个 **junction（目录传送门）**，程序以为还在 C，数据实际在 D。不删、不重装、不改配置。

## 什么时候用

- 用户说「C 盘满了 / 红了 / 不够用」。
- 想给某个大文件夹「搬家」，但不想改程序配置。
- 用户提到 junction / mklink / 目录链接 / 传送门。
- 已知大头：agent 家目录（`.codex`、`.workbuddy`、`.claude`）、缓存（`.cache`、`.ollama`、npm/pip）、node_modules、Docker/WSL。

## 铁律（fail-closed，必须遵守）

1. **先扫描，后动手。** 默认只出报告，**不移动、不删除任何东西**。
2. **移动前三确认**：目标盘空间够、相关程序**完全退出**（含托盘/后台）、重要数据已备份。
3. **建链只用 `/J`（junction）**，不用 `/D`（符号链接，需管理员/开发者模式，兼容差）。
4. **删链接只用 `rmdir "链接路径"`（不带 `/s`）**，禁止 `del /s`、资源管理器「删除并清空」。
5. **一次只搬一个目录**，搬完验证，再搬下一个。
6. **出错就回滚**：删链接 → 数据搬回原位 → 恢复程序。

## 预检（搬之前必跑，NO-GO 不许搬）

```powershell
powershell -ExecutionPolicy Bypass -File "<skill>\scripts\preflight_check.ps1" -Source "C:\Users\<你>\.codex" -Target "D:\AgentData\.codex"
```

自动检查：源存在且是**真文件夹**（非链接）、目标盘存在、目标非空、源/目标是否同盘、**目标空间是否 ≥ 源大小×1.1**。输出 `RESULT: GO` 或 `NO-GO`。

## 工作流

### 0. 扫描（只读）

```powershell
powershell -ExecutionPolicy Bypass -File "<skill>/scripts/scan_big_dirs.ps1"
```

输出：C 盘剩余空间 + 候选目录大小 / 是否已是 junction。**此步不做任何修改。**

### 1. 出方案

给用户一张表：搬哪个、搬到哪、预计腾出多少、风险点。

### 2. 等确认

**必须用户明确说「搬」才执行。** 不要自作主张。

### 3. 执行（逐个目录）

```bat
:: 0) 彻底退出程序（含托盘/后台服务）
:: 1) 搬
move "C:\Users\<你>\.codex" "D:\AgentData\.codex"
:: 2) 链（普通权限即可）
mklink /J "C:\Users\<你>\.codex" "D:\AgentData\.codex"
```

### 4. 验证

```bat
:: 应显示 <JUNCTION>
dir /a:l "C:\Users\<你>"
```
- 重开程序，登录态 / 历史 / 功能都在 → 成功。
- 失败 → 立即回滚。

### 5. 回滚

```bat
rmdir "C:\Users\<你>\.codex"
move "D:\AgentData\.codex" "C:\Users\<你>\.codex"
```
然后恢复程序。**回滚前确认 D 盘数据完好。**

## 脚本一览

| 脚本 | 作用 | 安全性 |
|---|---|---|
| `scan_big_dirs.ps1` | 列 C 盘空间 + 常见 AI 目录大小 / junction 状态 | 只读 |
| `preflight_check.ps1` | 搬前检查，输出 GO / NO-GO | 只读 |
| `move_with_junction.ps1` | 搬 + 建 junction；**默认干跑，`-Force` 才真执行** | 需确认 |
| `rollback_junction.ps1` | 删 junction + 数据搬回；需 `-Force` | 需确认 |

```powershell
# 1) 扫描
powershell -ExecutionPolicy Bypass -File "<skill>\scripts\scan_big_dirs.ps1"
# 2) 预检（换路径）
powershell -ExecutionPolicy Bypass -File "<skill>\scripts\preflight_check.ps1" -Source "C:\Users\<你>\.codex" -Target "D:\AgentData\.codex"
# 3) 干跑（不搬，只看计划）
powershell -ExecutionPolicy Bypass -File "<skill>\scripts\move_with_junction.ps1" -Source "C:\Users\<你>\.codex" -Target "D:\AgentData\.codex" -DryRun
# 4) 真搬（确认后）
powershell -ExecutionPolicy Bypass -File "<skill>\scripts\move_with_junction.ps1" -Source "C:\Users\<你>\.codex" -Target "D:\AgentData\.codex" -Force
```

## 决策表：哪些能搬

| 目标 | 例子 | 建议 |
|---|---|---|
| agent 家目录 | `.codex`、`.workbuddy`、`.claude` | ✅ 安全，优先搬 |
| 缓存 | `.cache`、`.ollama`、`.npm` | ✅ 安全 |
| Electron 数据 | `%APPDATA%\Codex` | ⚠️ 较敏感，先备份、先小样本 |
| node_modules | 项目内 | ⚠️ 删了重装更干净，不必搬 |
| Docker / WSL | `.vhdx` | ❌ 别用本法，走官方迁移 |
| 系统 / 程序目录 | `Windows`、`Program Files` | 🚫 禁止 |

## 常见目标

| 目标 | 典型路径 | 说明 |
|---|---|---|
| agent 家目录 | `~/.codex`、`~/.workbuddy`、`~/.claude` | 配置/日志/记忆/沙箱，通常几百 MB~数 GB |
| 缓存 | `~/.cache`、`~/.ollama`、`~/.npm` | 模型与包缓存，往往最大 |
| Electron 数据 | `%APPDATA%\<App>` | 缓存/localStorage/GPU cache，较敏感，先小样本试 |
| node_modules | 项目内 | 可直接删了重装，未必值得搬 |
| Docker/WSL | `%LOCALAPPDATA%\Docker`、WSL vhdx | 体积大，但迁移方式不同，勿硬搬 |

## 坑（详见 references/junction-pitfalls.md）

| 坑 | 正确做法 | 后果 |
|---|---|---|
| 程序还开着就搬 | 先退出（含后台） | 文件占用，搬不动 / 损坏 |
| 用 `/D` 建符号链接 | 用 `/J` 建 junction | 需管理员权限，兼容差 |
| 用 `del /s` 删链接 | `rmdir "链接"`（不带 /s） | 顺着链接删掉目标盘真数据 |
| 目标指网络盘 | 只指本地盘（NTFS/exFAT 均可） | junction 不支持网络路径 |
| 升级/重装后 | 检查是否又在 C 重建目录 | 需要重做 junction |

### 脚本编码（踩过的坑）

`.ps1` 正文**必须纯 ASCII**。PowerShell 5.1 读取无 BOM 的 UTF-8 文件时按 ANSI 解析，中文注释/字符串会变乱码，直接 `ParserError`。
SKILL.md 用中文没问题（Markdown 不经过 PowerShell 解析）；只有 `.ps1` 要注意。

## 排障表

| 症状 | 可能原因 | 处理 |
|---|---|---|
| `mklink` 报「已存在」 | 原位还有空文件夹 | 先删空文件夹再建链 |
| 建链后程序打不开 / 丢配置 | 程序不认 junction，或搬时没关干净 | 用 `rollback_junction.ps1` 回滚，再排查 |
| 搬一半报错 | 文件被占用 / 权限不足 | 关程序重试；robocopy 已保留源，不会丢数据 |
| 用 `del /s` 后 D 盘数据没了 | 顺着 junction 删了目标 | 立刻停手，用数据恢复；以后只用 `rmdir`（不带 /s） |
| 升级后 C 盘又涨 | 程序重建了家目录 | 检查并重做 junction |
| 目标盘是 exFAT | 可用，但丢 ACL/权限 | 无需换盘；只要链接建在 NTFS |

## 边界（诚实交代）

- **仅 Windows**：链接必须建在 NTFS 盘（源盘）；目标可为本地 exFAT（**本机实测可用**），但会丢 ACL/权限；mac/Linux 用 symlink。
- **部分 Electron 应用对 junction 敏感**，搬前备份、先拿小的试。
- **Docker/WSL 不要用本法硬搬**，走各自的迁移流程。
- junction 的链接必须建在 **NTFS** 盘；目标可为本地其他盘（**exFAT 实测可用**），但不能指网络盘。

## 验证清单（交付前必过）

- [ ] `dir` 确认原位是 `<JUNCTION>`。
- [ ] 重开程序，功能正常、数据在。
- [ ] C 盘空间确实腾出。
- [ ] 记下回滚命令，出问题能还原。
