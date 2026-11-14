# 一页纸命令卡

> 复制即用。`$d` 先设成出问题的目录。

```powershell
$d = "$env:USERPROFILE\Desktop"
```

---

## 定位（3 条命令内出结论）

```powershell
# 1. 强制标签 —— 核心证据
icacls $d | Select-String "Mandatory"

# 2. 我自己的完整性级别 —— 决定我的实验可不可信
whoami /groups | Select-String "Mandatory Label"

# 3. 是否真有下载标记（排除 MOTW 假象）
Get-ChildItem $d -Recurse -File -ErrorAction SilentlyContinue |
  Where-Object { Get-Item $_.FullName -Stream Zone.Identifier -ErrorAction SilentlyContinue } |
  Measure-Object | Select-Object -ExpandProperty Count
```

| 看到什么 | 结论 |
|---|---|
| `Low Mandatory Level:(OI)(CI)(NW)` | **就是它**，去修复 |
| `Medium Mandatory Level` 或无标签 | 不是这个问题，换方向 |
| `whoami` 里自己也是 `Low` | 你的复现实验全部无效，换外部进程 |
| 标记数量 = 0 或只占少数 | 排除「文件被标记」路线 |

---

## 取证（写报告/提 issue 用）

```powershell
# 完整 ACL：Deny 项、合成 SID、继承状态
icacls $d

# 父目录 + 子项，确认递归继承范围
icacls (Split-Path $d) ; icacls $d

# 所有者的完整性标签（同上，换角度）
Get-Acl $d | Format-List

# 本地文件的区域判定（确认 file:// 的 Zone 归属）
# 输出 3 = Internet 区域，这是 Windows 默认行为，不是异常
```

---

## 修复（管理员终端）

```powershell
# 把整棵树恢复为中完整性
icacls $d /setintegritylevel "(OI)(CI)M" /T /C

# 被拒绝时：先夺所有权 + 重置 ACL，再改标签
takeown /f $d /r /d y
icacls $d /reset /T /C
icacls $d /setintegritylevel "(OI)(CI)M" /T /C

# 让外壳按新标签重建（资源管理器自动重启）
Stop-Process -Name explorer -Force
```

---

## 验证（必须在外部进程里做）

```powershell
# 标签已恢复
icacls $d | Select-String "Mandatory"      # 不应再有 Low

# 功能性验证：去资源管理器里双击一个文件、把文件拖进该目录
```

---

## 常见误判对照

| 误判 | 事实 |
|---|---|
| 「Unblock-File 能修好」 | 若标签是 Low，清标记毫无作用；弹窗文案里的 "Internet" 是外壳的措辞 |
| 「`file = 3` 是元凶」 | `file://` 映射到 Internet 区域是 Windows 默认 |
| 「`ScanWithAntiVirus = 3` 是元凶」 | 删掉它症状不变，就不是它 |
| 「我测了，能写，所以没问题」 | 你也在 Low 令牌里，同级写当然通过 |
| 「权限正常，所以不是权限问题」 | DACL 与强制标签是两套机制，DACL 正常不影响 `(NW)` 生效 |

---

## 风险提示

`/setintegritylevel ... /T /C` 会**递归**改目标树下所有对象。不要对着 `C:\Windows`、
`C:\Program Files`、网络共享或含大量受管文件的目录执行。先在小目录演练，或先备份 ACL：

```powershell
icacls $d /save "$env:TEMP\acl-backup.txt" /T
```
