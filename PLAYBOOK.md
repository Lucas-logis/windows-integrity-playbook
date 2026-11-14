# 排查预案：目录被降为低完整性（Low IL + No-Write-Up）

> 适用于：**某一批文件的写操作集体失效 + 伴随「可能有害 / Internet 安全设置」类提示**。
> 本预案从一次真实事故抽象而来，可用在任何「某个目录树突然变只读、且系统在警告」的场合。

---

## 0. 触发条件（先判断值不值得走这个流程）

命中 2 条以上再往下走：

| # | 观察到的现象 |
|---|---|
| 1 | 同一目录树下，打开 / 移动 / 复制 / 重命名 / 删除**集中**失败，报「访问被拒绝」 |
| 2 | 双击文件或快捷方式弹安全警告，文案含「Internet 安全设置」「可能有害」「不受信任」 |
| 3 | 该目录曾作为某个沙箱化工具（AI agent、容器、虚拟机共享目录、应用沙箱）的工作区 |
| 4 | 目录 `icacls` 输出里出现 `Mandatory Label` 且带 `Low` |

---

## 1. 判定树

```
某类操作全面失灵 + 安全提示
│
├─ ① icacls <目录> | Select-String Mandatory
│    │
│    ├─ 出现 Low  → ★ 本条路径：低完整性降级，跳到 §3 修复
│    │
│    └─ 没有 Low
│         │
│         ├─ ② 有没有 Zone.Identifier（真实下载标记）？
│         │    ├─ 有，且集中在少量文件 → 附件管理器 / MOTW 路线，Unblock-File
│         │    └─ 没有 → 往下
│         │
│         ├─ ③ 安全软件是否有外壳钩子 / 隔离记录？
│         │    ├─ 有 → 查隔离区与 HIPS 日志
│         │    └─ 没有 → 往下
│         │
│         └─ ④ 目录 DACL 是否异常（Deny 项、所有者错误、继承被断）？
│              ├─ 有 → icacls /reset 或重设所有权
│              └─ 没有 → 抓 ProcMon（shell32 / Explorer）看判定过程
```

> 顺序很重要：**先看强制标签**，因为它是唯一「一眼可判定、且修复成本最低」的证据。
> 事故里的我反过来了——先查标记、再查注册表、最后才看标签，绕了 40 分钟。

---

## 2. 取证（只读，不改任何东西）

```powershell
# 目标目录
$d = "$env:USERPROFILE\Desktop"

# ① 强制标签（核心证据）
icacls $d | Select-String "Mandatory"

# ② 看完整 ACL：Deny 项、合成 SID（S-1-4-*）、继承是否被切断
icacls $d

# ③ 看一个子项，确认标签是否递归继承
(Get-ChildItem $d -Directory | Select-Object -First 1).FullName | ForEach-Object { icacls $_ } | Select-String "Mandatory"

# ④ 当前进程自己的完整性级别（重要：用来判断「我能不能做这个实验」）
whoami /groups | Select-String "Mandatory Label"

# ⑤ 排除 MOTW：真的带下载标记的文件有几个
Get-ChildItem $d -Recurse -File -ErrorAction SilentlyContinue |
  Where-Object { Get-Item $_.FullName -Stream Zone.Identifier -ErrorAction SilentlyContinue } |
  Measure-Object | Select-Object Count
```

判定要点：

- `Low Mandatory Level:(OI)(CI)(NW)` → `Low` 是级别，`(OI)(CI)` 是会向下继承，`(NW)` 是禁止向上写；
- DACL 里 `FullControl` 齐全、却仍然写不进去 → 几乎可以断定是强制标签，不是权限问题；
- 如果 `whoami /groups` 显示你自己的令牌也是 `Low`，**你的所有复现实验都不可信**（见 §5）。

---

## 3. 修复

前提：**必须用中完整性或更高权限的进程**（管理员终端 / 资源管理器）。

```powershell
# 常规修复：把目录树整体恢复为中完整性
icacls "$env:USERPROFILE\Desktop" /setintegritylevel "(OI)(CI)M" /T /C
```

参数：`/setintegritylevel` 改强制标签、`(OI)(CI)` 递归继承、`M` = Medium、`/T` 递归、`/C` 遇错继续。

若报「拒绝访问」（标签被锁死或所有者不对）：

```powershell
takeown /f "$env:USERPROFILE\Desktop" /r /d y
icacls "$env:USERPROFILE\Desktop" /reset /T /C
icacls "$env:USERPROFILE\Desktop" /setintegritylevel "(OI)(CI)M" /T /C
```

修复后：

```powershell
# 让图标缓存 / 外壳状态按新标签重建
Stop-Process -Name explorer -Force   # 资源管理器会自动重启；或在任务管理器里「重新启动」
```

> 顺序建议：先修标签 → 再重启资源管理器。反过来会让图标先按旧标签重建一遍，白改一次。

---

## 4. 验证（必须用外部进程，不能只靠自测）

| 验证项 | 方法 | 期望 |
|---|---|---|
| 标签已恢复 | `icacls <目录> \| Select-String Mandatory` | 不再出现 `Low` |
| 子项已恢复 | 抽查 2–3 个层级 | 同上 |
| 真实故障消失 | **在资源管理器里**双击原文件 / 把文件拖入该目录 | 无弹窗、能写入 |
| 图标恢复 | 重启资源管理器后观察 | 快捷方式图标正常 |

> 「我在终端里测通过了」不算验证通过——见 §5。

---

## 5. 最大的坑：在受污染的进程里验证

如果故障的成因是「这个目录被降级为 Low」，而你的排查命令本身也跑在 Low 令牌下，那么：

- 你写这个目录 = 同级写 = **不被 `(NW)` 阻止** → 测试通过；
- 真实的用户进程（资源管理器，中完整性）= 向上写 = 被拒绝 → 故障依旧。

**结论：测试环境必须与被测对象隔离。** 用资源管理器、管理员终端、或另开一个未受管的普通进程来验证。

---

## 6. 症状 → 根因对照表

| 症状 | 单一根因（Low IL + NW）能解释吗 | 说明 |
|---|---|---|
| 双击 .lnk / .exe 弹「可能有害」 | 能 | 低完整性内容被外壳视为不受信任来源 |
| 打开文件夹（不进入）正常 | 能 | 读不需要向上写 |
| 文件移不进、复制报错 | 能 | 中完整性进程向 Low 对象写入 |
| 快捷方式图标变白板 | 能（间接） | 图标缓存读取/更新受限，退回默认图标 |
| 只有 .lnk 弹、.png/.txt 不弹 | 能 | 附件管理器按文件类型风险分级，快捷方式优先撞上 |
| 打开文件本身报「损坏」 | **不能** | 换方向：文件系统 / 磁盘 / 应用关联 |
| 整个磁盘都写不进去 | **不能** | 换方向：卷只读、BitLocker、磁盘错误 |

---

## 7. 防复发

| 措施 | 说明 |
|---|---|
| 专用工作目录 | `D:\Agent-Work` 之类，不和桌面/文档混用；在工具里单独注册为工作区 |
| 桌面只放壳 | 快捷方式 + 文档，不参与沙箱化执行 |
| 修复类操作先进低权限环境演练 | `icacls /setintegritylevel` 会递归，先在小目录试 |
| 变更留痕 | 改注册表/ACL 前导出备份，改完不复现就还原 |
| 升级后复查 | 工具升级、换安装路径后，安全软件信任区条目需要重新添加 |

---

## 8. 上报

如果确认是某个工具（而不是用户操作）主动降级了用户目录，这属于该工具的**设计缺陷**，值得提 issue：

- 工作区根目录的隔离状态**不应污染已存在的用户文件**；
- 降级应只作用于工具自己创建的临时目录，或使用不继承的 `(OI)` 之外策略；
- 退出时应能自动还原标签。

上报时附上：`icacls` 原始输出、复现步骤、工具版本、以及「修复后症状消失」的对照结果——这套证据链本仓库的 [case-study](case-study/) 就是现成模板。
