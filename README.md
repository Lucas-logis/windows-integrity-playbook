# 一次 Windows 低完整性事故，和它换来的排查手册

> **English summary.** A desktop AI agent tool lowered the integrity level of my
> Desktop folder to *Low* with *No-Write-Up* (`icacls` → `Low Mandatory Level:(OI)(CI)(NW)`).
> Windows then treated every file on the Desktop as untrusted: double-clicking anything
> raised a "these files might be harmful" warning, files could not be moved onto the
> Desktop, and shortcut icons turned blank. It took **56 minutes** to find the cause —
> and roughly 40 of those minutes were spent in the wrong direction. This repository is
> the playbook I wish I had on minute one, plus the unedited forensic trail of the case
> (including every dead end I walked into).

---

## 一句话

弹窗说的是「Internet 安全设置」，真凶却是弹窗里从没出现的一个词：**完整性级别（Integrity Level）**。

| | |
|---|---|
| **症状** | 桌面上的文件和快捷方式双击弹「Windows 安全中心：打开这些文件可能会对你的计算机有害」；文件移不进桌面；快捷方式图标变成白板 |
| **真凶** | 桌面目录的强制标签：`Mandatory Label\Low Mandatory Level:(OI)(CI)(NW)` |
| **一条命令定位** | `icacls "$env:USERPROFILE\Desktop" \| Select-String Mandatory` |
| **一条命令修复** | `icacls "$env:USERPROFILE\Desktop" /setintegritylevel "(OI)(CI)M" /T /C`（需管理员） |
| **本次耗时** | 56 分钟到根因；其中约 40 分钟被错误方向消耗 |

![Windows 安全中心拦截弹窗](case-study/evidence/windows-security-block.png)

---

## 30 秒应急

```powershell
# 1. 先看强制标签 —— 出现 Low 就基本结案
icacls "$env:USERPROFILE\Desktop" | Select-String "Mandatory"

# 2. 确认是 Low 之后，以【管理员】身份修复
icacls "$env:USERPROFILE\Desktop" /setintegritylevel "(OI)(CI)M" /T /C

# 3. 重启资源管理器（或重启系统），让图标缓存按新完整性重建
```

判断口诀：

- `Mandatory Label` 里出现 **`Low`** → 就是它，直接修，别再查下载标记；
- 没有 `Low`、也没有 `Zone.Identifier` → 换方向：附件管理器策略 / 安全软件外壳钩子；
- **只在外部进程里复现**（资源管理器、管理员终端），不要在受管工具自己的沙箱里验证。

---

## 为什么值得一读

这不是一篇「Low IL 是什么」的科普，而是三件更贵的东西：

1. **一个可复用的判定流程。** 见 [PLAYBOOK.md](PLAYBOOK.md)：从症状矩阵 → 判定树 → 修复 → 验证 → 防复发，脱离本次事故也成立。
2. **一次完整的事故取证。** 见 [case-study/](case-study/)：安全报告、技术分析、逐分钟试错日志，证据链、置信度、副作用还原全都留着。
3. **一份诚实的复盘。** 见 [RETROSPECTIVE.md](RETROSPECTIVE.md)：为什么我在同一个错误方向上耗掉 40 分钟——锚定效应、用被污染的进程做验证、以及绕开最直接的元数据。

> 记录无效尝试的价值，往往高于记录结论：结论别人也能写，弯路只有走过的人知道。

---

## 仓库导航

| 文件 | 内容 | 什么时候看 |
|---|---|---|
| [PLAYBOOK.md](PLAYBOOK.md) | 通用排查预案：症状 → 假设 → 验证 → 修复 → 防复发 | 现在就遇到了类似问题 |
| [CHEATSHEET.md](CHEATSHEET.md) | 一页纸命令卡（定位 / 取证 / 修复 / 验证） | 想直接抄命令 |
| [RETROSPECTIVE.md](RETROSPECTIVE.md) | 方法论复盘：认知偏差、验证环境谬误、可迁移的经验 | 关心「怎么排查」而不只是「怎么修」 |
| [case-study/00-事件摘要.md](case-study/00-事件摘要.md) | 本次事故一页速览 | 先看个大概 |
| [case-study/01-安全报告.md](case-study/01-安全报告.md) | 影响面、根因、证据链、风险评级、处置 | 想知道「到底发生了什么」 |
| [case-study/02-技术分析.md](case-study/02-技术分析.md) | IL 与 DACL 的区别、症状推导、三个假象的排除、置信度 | 想搞懂「为什么」 |
| [case-study/03-试错日志.md](case-study/03-试错日志.md) | 逐分钟时间线、无效尝试清单、副作用与清理 | 想避免重走弯路 |
| [templates/](templates/) | 空白模板：事件摘要 / 安全报告 / 技术分析 / 试错日志 | 想给自己的事故建同样的档案 |

---

## 适用的症状（快速自检）

三条里中两条以上，就值得先跑一遍 `icacls`：

- [ ] 对**某一批**文件的全套写操作（打开、移动、复制、重命名）集中失效，报错多为「访问被拒绝」；
- [ ] 同时伴随安全提示，文案提到「Internet 安全设置」「可能有害」「不受信任的来源」；
- [ ] 这些文件都在**同一个目录树**下——尤其如果你曾把桌面、文档目录交给某个带沙箱的 AI 工具当工作区；
- [ ] `icacls` 的输出里出现 `Low Mandatory Level`（决定性一条）。

---

## 本次事故的关键数字

| 数据 | 含义 |
|---|---|
| `Low Mandatory Level:(OI)(CI)(NW)` | 唯一真凶：低完整性 + 禁止向上写 + 递归继承 |
| 7,026 个文件 / 131 个下载标记 | 标记全部来自真实下载；与本次故障无关 |
| 0 个受管工具生成的文件带标记 | 彻底排除「工具给文件打标记」这条线 |
| 1 条 `icacls` 命令 | 真正的修复成本 |

---

## 防复发（比修复更重要）

1. **不要把桌面、文档、下载这类日常目录交给带沙箱的工具当工作区。**
2. 新建专用目录（例如 `D:\Agent-Work`），在工具里单独注册成工作区。
3. 桌面只放快捷方式和文档，不参与任何沙箱化执行。
4. 工具升级或更换安装路径后，安全软件的信任区条目需要重新添加。

---

## 范围与免责

- 本文所有命令针对 **Windows 10/11** 的 NTFS + UAC 完整性机制；不适用于非 NTFS 卷。
- `icacls ... /setintegritylevel "(OI)(CI)M" /T /C` 会**递归**改写目标目录树下所有对象的标签。
  对个人目录通常是恢复默认，但**动手前请先备份或在无关目录上演练**，不要直接对着 `C:\Windows`、`C:\Program Files` 或网络共享执行。
- 本仓库为个人学习记录，非官方文档；产品行为描述以我观测到的版本为准，厂商可能已在新版本中修复。

---

## 致谢

微软 Q&A 上那则同症状问答给了决定性的方向提示（`icacls /setintegritylevel M`），是本次排查的转折点。

## License

[MIT](LICENSE)
