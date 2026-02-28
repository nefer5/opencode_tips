---
id: note-003
type: note
title: "Agent防误删围栏方案：三级防护机制"
published: "2026-02-28"
status: draft
tags: [安全, 文件操作, 删除保护, 回收站, windows, agent规范]
---

# Agent防误删围栏方案：三级防护机制

## 1. 背景：要解决的问题

### 1.1 问题场景

在Agent文件操作过程中，存在误删用户重要文件的风险。当用户要求删除文件/目录时，Agent可能因以下原因造成不可逆损失：

| 风险类型 | 典型表现 | 后果 |
|---------|---------|------|
| 路径误输入 | 用户手滑输错路径 | 删错目录 |
| 理解偏差 | Agent误解用户意图 | 删除范围过大 |
| 批量操作失控 | 删除包含多个文件的目录 | 误删重要文件 |
| 永久删除误触发 | rm命令误执行 | 数据永久丢失 |

### 1.2 问题根源

1. **Agent默认使用rm命令**：会直接永久删除文件，无回收机制
2. **用户不知情**：用户可能不清楚删除的具体影响范围
3. **缺少确认机制**：批量操作时未进行二次确认
4. **安全意识不足**：Agent未被明确要求执行安全删除流程

### 1.3 触发条件

当Agent需要执行文件/目录删除操作时，应该启用防误删围栏机制。特别是以下场景：

| 条件 | 风险等级 | 说明 |
|------|---------|------|
| 删除目录 | 高 | 目录可能包含多个子文件 |
| 目标包含≥3个文件 | 极高 | 批量删除风险 |
| 用户要求"永久删除" | 极高 | 不可恢复 |
| 涉及用户主目录 | 极高 | 可能包含重要配置 |

---

## 2. 解决思路：关键逻辑

### 2.1 核心设计理念

采用**"三级防护"**策略，从技术和流程两个层面建立安全屏障：

```
┌─────────────────────────────────────────────────────────┐
│                    防误删围栏体系                         │
├─────────────────────────────────────────────────────────┤
│  第一层：技术防护                                          │
│  ├── 禁止直接使用rm命令                                    │
│  └── 强制使用回收站脚本（可恢复）                           │
├─────────────────────────────────────────────────────────┤
│  第二层：流程防护                                          │
│  ├── ≥3个文件：强制用户确认                                 │
│  └── 等待明确yes/no后再执行                                 │
├─────────────────────────────────────────────────────────┤
│  第三层：意识防护                                          │
│  ├── 永久删除前提醒备份（手动/git）                          │
│  └── 二次确认（question工具）                               │
└─────────────────────────────────────────────────────────┘
```

### 2.2 关键决策逻辑

```
开始删除操作
    │
    ▼
用户是否要求"永久删除"? 
    │
    ├── 是 → 提醒备份(git/手动) → 二次确认(question) → 执行rm
    │
    └── 否 → 使用回收站脚本
                │
                ▼
        目标包含≥3个文件?
                │
                ├── 是 → 列出文件清单 → 强制确认(question) → 执行recycle.ps1
                │
                └── 否 → 直接执行recycle.ps1
```

---

## 3. 落地实施

### 3.1 实施方式

通过 **AGENTS.md规范** + **PowerShell回收站脚本** 实现，将安全删除要求写入Agent提示词，强制遵守。

**重要提醒**：本方案基于Windows 11环境开发，其他系统需要自行适配回收站脚本。

### 3.2 文件组织结构

```
项目根目录/
├── .opencode/
│   └── scripts/
│       └── recycle.ps1      # PowerShell回收站脚本（Windows）
├── AGENTS.md                # 包含防误删围栏提示词
└── ...
```

### 3.3 完整代码/配置

#### 1. AGENTS.md中的防误删围栏提示词

在AGENTS.md的"环境与操作规范"章节添加以下内容（可根据自己需求微调）：

```markdown
### 防误删围栏 (重要!)

- **禁止直接rm**: 所有删除操作必须通过回收站脚本 `.opencode\scripts\recycle.ps1` 执行
- **≥3个文件强制确认**: 目标包含≥3个文件时，**即使移入回收站**也必须使用 `question` 工具获取用户 yes/no 确认，**等待输入后再执行**
- **强制rm二次确认**: 用户明确要求彻底删除时，必须先提醒备份（手动或git），再使用 `question` 工具获取 yes/no 确认
```

#### 2. PowerShell回收站脚本（Windows专用）

创建文件 `.opencode/scripts/recycle.ps1`：

```powershell
param(
    [Parameter(Mandatory=$true)]
    [string]$Path
)

# 检查路径是否存在
if (-not (Test-Path -LiteralPath $Path)) {
    Write-Error "Path not found: $Path"
    exit 1
}

$item = Get-Item -LiteralPath $Path
$fullPath = $item.FullName

# 加载VisualBasic程序集（用于回收站功能）
try {
    Add-Type -AssemblyName Microsoft.VisualBasic
}
catch {
    Write-Error "Failed to load Microsoft.VisualBasic"
    exit 1
}

# 执行删除操作（移至回收站）
try {
    if ($item.PSIsContainer) {
        # 删除目录到回收站
        [Microsoft.VisualBasic.FileIO.FileSystem]::DeleteDirectory(
            $fullPath,
            [Microsoft.VisualBasic.FileIO.UIOption]::OnlyErrorDialogs,
            [Microsoft.VisualBasic.FileIO.RecycleOption]::SendToRecycleBin
        )
        Write-Host "Moved directory to Recycle Bin: $fullPath"
    }
    else {
        # 删除文件到回收站
        [Microsoft.VisualBasic.FileIO.FileSystem]::DeleteFile(
            $fullPath,
            [Microsoft.VisualBasic.FileIO.UIOption]::OnlyErrorDialogs,
            [Microsoft.VisualBasic.FileIO.RecycleOption]::SendToRecycleBin
        )
        Write-Host "Moved file to Recycle Bin: $fullPath"
    }
}
catch {
    Write-Error "Recycle operation failed: $($_.Exception.Message)"
    exit 1
}
```

### 3.4 使用方式

#### 场景1：普通删除（移入回收站）

```powershell
# 命令格式
powershell .opencode/scripts/recycle.ps1 "目标路径"

# 示例
powershell .opencode/scripts/recycle.ps1 "D:\MyProject\temp\old_files"
```

#### 场景2：删除包含≥3个文件的目标

**必须**先使用`question`工具获取用户确认：

1. 列出目标内容：`ls -la "目标路径"`
2. 使用`question`工具询问是否确认删除
3. 等待用户明确回答后再执行

#### 场景3：用户要求永久删除

1. 提醒用户备份（建议git提交或手动复制）
2. 使用`question`工具获取明确确认
3. 确认后才可使用`rm`命令

#### macOS/Linux用户适配建议

如果你的系统不是Windows，需要自行实现回收站功能：

- **macOS**：可使用`trash`命令（需安装`brew install trash`）
- **Linux**：可使用`gio trash`或`trash-cli`工具

示例适配脚本（macOS）：
```bash
#!/bin/bash
# macOS回收站脚本 .opencode/scripts/recycle.sh
if [ -z "$1" ]; then
    echo "Usage: $0 <path>"
    exit 1
fi

if [ ! -e "$1" ]; then
    echo "Path not found: $1"
    exit 1
fi

trash "$1"
echo "Moved to Trash: $1"
```

---

## 4. 其他补充

### 4.1 推荐使用方式

**建议根据自己的系统环境调整适配**：

- **Windows用户**：可直接使用上述PowerShell脚本，无需修改
- **macOS/Linux用户**：需要替换回收站脚本，但AGENTS.md中的提示词逻辑可保留
- **所有用户**：建议根据自己项目的实际目录结构调整脚本存放位置

### 4.2 关键坑点

#### 坑点1：脚本路径不匹配

**问题**：AGENTS.md中提示的路径`.opencode\scripts\recycle.ps1`与实际存放位置不一致  
**解决**：确保脚本实际存放位置与AGENTS.md中提示的路径一致

#### 坑点2：系统环境差异

**问题**：PowerShell脚本仅适用于Windows，其他系统无法直接使用  
**解决**：根据你的操作系统，选择对应的回收站实现方案（见3.4节）

#### 坑点3：Agent未读取AGENTS.md

**问题**：Agent可能没有正确读取AGENTS.md中的提示词，导致未执行防误删流程  
**解决**：
- 方案A：在对话开头明确提醒Agent"请遵守防误删围栏规则"
- 方案B（进阶）：可创建一个专门的"删除文件"Skill，作为双重保障
  - ⚠️ **注意**：Skill触发也有小概率命中不了，不能完全依赖

#### 坑点4：Git Bash路径问题

**问题**：OpenCode的bash工具底层使用Git Bash，路径格式可能与PowerShell不兼容  
**解决**：使用绝对路径调用脚本，确保路径格式正确

### 4.3 验收标准

- [ ] 已创建回收站脚本（`.opencode/scripts/recycle.ps1`或对应系统的版本）
- [ ] AGENTS.md中已添加防误删围栏提示词
- [ ] 测试删除包含3个以上文件的目录时，Agent会主动询问确认
- [ ] 测试要求永久删除时，Agent会提醒备份并询问确认
- [ ] 测试普通删除时，文件确实进入回收站（而非永久删除）

### 4.4 扩展阅读

**相关概念**：
- [Microsoft.VisualBasic.FileIO命名空间](https://docs.microsoft.com/en-us/dotnet/api/microsoft.visualbasic.fileio) - 回收站API文档
- [Git备份最佳实践](https://git-scm.com/doc) - 使用git保护重要数据

**类似方案**：
- 可结合`safe-fast-edit`Skill，在批量编辑前创建备份
- 可考虑使用文件系统快照（如Windows卷影复制）作为额外保护

**进阶防护**：
如果你担心Agent没有正确读取AGENTS.md提示词，可以考虑：
1. 创建一个`delete-file`Skill，将删除逻辑封装为技能
2. 在Skill中强制实现三级防护逻辑
3. 但注意Skill触发也有小概率失效，最好同时保留AGENTS.md提示词

---

*最后更新：2026-02-28*  
*作者：雅珂达 (Monarch)*  
*适用环境：Windows 11 + OpenCode + PowerShell*
