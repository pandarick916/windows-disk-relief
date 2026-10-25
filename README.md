# windows-disk-relief

> 把 Windows C 盘上的大文件夹（AI 工具的家目录、模型缓存等）搬到别的盘，程序照常使用。
> Move large folders off your C: drive and keep apps working, via directory junctions.

---

## 这是什么（一句话）

很多 AI 工具（Codex、Claude、WorkBuddy、Ollama 等）会把配置、日志、模型、缓存塞进 C 盘。
本工具把它们**搬到 D 盘（或别的盘）**，再在原位置放一个「传送门（junction）」——
程序以为文件还在 C 盘，其实数据已经在 D 盘。**不删、不重装、不改设置。**

## 适合谁

- ✅ C 盘红了，被 AI 工具吃掉几十 G
- ✅ 想搬 `~/.codex`、`~/.cache`、`~/.ollama` 这类文件夹
- ❌ Docker / WSL 的虚拟磁盘（请用官方迁移）
- ❌ 系统目录（Windows、Program Files）——脚本会直接拒绝

---

## ⚠️ 先看这三条（安全）

1. **搬之前，先关掉相关程序**（托盘、后台也退）。
2. **重要数据先备份。**
3. **删「传送门」只能用 `rmdir`，绝不能用 `del /s`**（会顺着门删掉 D 盘的真数据）。

---

## 3 步开始（照着做就行）

### 第 1 步：扫描，看谁最胖

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\scan_big_dirs.ps1
```

列出 C 盘剩余空间 + 常见 AI 目录的大小。也可以只扫一个目录：

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\scan_big_dirs.ps1 -Paths "$env:USERPROFILE\.codex"
```

### 第 2 步：预检，确认能搬

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\preflight_check.ps1 -Source "$env:USERPROFILE\.codex" -Target "D:\AgentData\.codex"
```

输出 `RESULT: GO` 才能继续；`NO-GO` 就先别搬。

### 第 3 步：干跑 → 真搬

```powershell
# 干跑：只看计划，不动手
powershell -ExecutionPolicy Bypass -File .\scripts\move_with_junction.ps1 -Source "$env:USERPROFILE\.codex" -Target "D:\AgentData\.codex" -DryRun

# 确认没问题后，真搬（记得先关程序）
powershell -ExecutionPolicy Bypass -File .\scripts\move_with_junction.ps1 -Source "$env:USERPROFILE\.codex" -Target "D:\AgentData\.codex" -Force
```

---

## 出问题了？一键回滚

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\rollback_junction.ps1 -Source "$env:USERPROFILE\.codex" -Target "D:\AgentData\.codex" -Force
```

---

## 把这个项目丢给 AI agent

把整个文件夹交给 agent（Claude Code / Codex / WorkBuddy / Cursor 都行），说一句：

> 「读 SKILL.md，帮我把 `~/.codex` 搬到 D 盘。先预检、干跑，我确认后再搬。」

agent 会按 `SKILL.md` 的流程走：**扫描 → 预检 → 干跑 → 等你确认 → 执行 → 校验**。
默认不会动手，必须你点头。

---

## 文件结构

```
windows-disk-relief/
├─ SKILL.md                     # 给 agent 看的说明书（触发词、流程、铁律）
├─ README.md                    # 你正在看的这个
├─ LICENSE
├─ scripts/
│  ├─ scan_big_dirs.ps1         # 只读扫描
│  ├─ preflight_check.ps1       # 只读预检，出 GO/NO-GO
│  ├─ move_with_junction.ps1    # 搬 + 建 junction（默认干跑）
│  └─ rollback_junction.ps1     # 回滚
└─ references/
   └─ junction-pitfalls.md      # 三个坑 + 真实案例
```

---

## 常见问题

**Q：junction 和快捷方式一样吗？**
不一样。快捷方式是个文件；junction 是文件系统级的「重解析点」，程序完全感知不到。

**Q：为什么不用 symlink？**
symlink 更松（能指网络、相对路径），但要管理员权限。搬本地目录，junction 够用且免提权。

**Q：D 盘是 exFAT 可以吗？**
可以。但 exFAT 不存 ACL，权限会丢——个人缓存 / agent 数据通常无所谓。

**Q：会删我的数据吗？**
脚本只「移动」，不删。回滚能把数据搬回去。唯一危险是**手动用 `del /s` 删传送门**——别这么干。

**Q：需要管理员权限吗？**
不需要。junction 普通权限就能建。

---

## License

MIT。详见 [LICENSE](LICENSE)。

---

## English summary

Move large folders (AI agent homes like `~/.codex`, model caches, etc.) off the Windows C: drive and replace them with a **directory junction**, so apps keep working without any config change.

- **Safety first**: close the app, back up, and never delete a junction with `del /s` — use `rmdir`.
- **Workflow**: `scan → preflight → dry-run → move → verify → rollback`.
- **No hardcoded paths**: everything is passed via `-Source` / `-Target`.
- **Windows only**; the junction link must live on NTFS. The target may be exFAT (ACLs are not preserved).
