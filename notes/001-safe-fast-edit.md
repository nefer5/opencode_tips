---
id: note-001
type: note
title: "大文件编辑安全协议（safe-fast-edit）"
published: "2026-02-23"
status: draft
tags: [workflow, editing, safe-fast-edit, tools, 批量编辑, 文件安全]
---

# 大文件编辑安全协议（safe-fast-edit）

## 1. 背景：要解决的问题

### 1.1 问题场景

在使用 OpenCode 的 `edit` 工具进行大文件批量修改时，频繁遇到以下问题：

| 问题类型 | 典型表现 | 发生场景 |
|---------|---------|---------|
| **竞态问题** | `line changed` 报错 | 多轮对话中文件已被修改，行号对应不上 |
| **匹配歧义** | `text not found` 报错 | 文件内容已被其他工具修改，锚点丢失 |
| **重复插入** | 相同内容被插入多次 | 使用 `insert_after` 时未检查是否已存在 |
| **行号漂移** | 修改后后续操作错位 | 文件行数变化导致后续 edit 操作位置错误 |

### 1.2 问题根源

`edit` 工具基于**行号定位**，存在以下固有缺陷：

1. **行号敏感**：`edit` 工具使用行号作为锚点，一旦文件内容变化，行号即失效
2. **单次提交**：每次 `edit` 是独立操作，失败不会自动回滚已成功的修改
3. **上下文不足**：简单的文本匹配容易因重复内容而错位
4. **无预校验**：修改前无法验证是否会成功，失败后文件处于半修改状态

### 1.3 触发条件

当满足以下任一条件时，**必须**切换到安全批量编辑模式：

| 条件 | 阈值 | 说明 |
|------|------|------|
| `edit` 连续失败次数 | ≥2 次 | 典型报错：`line changed` / `text not found` / `匹配歧义` |
| 单文件修改点 | ≥3 处 | 且目标文件 >300 行 |

---

## 2. 解决思路：关键逻辑

### 2.1 核心设计理念

**唯一锚点 + 批量原子提交**：

1. **唯一锚点**：SEARCH 文本必须包含足够的上下文（至少 2 行），确保在全文唯一
2. **批量原子提交**：所有修改在内存中完成，预校验通过后再一次性写入
3. **预校验机制**：写入前检查 SEARCH 唯一性和块间冲突
4. **失败即终止**：任何校验失败立即停止，原文件保持不变

### 2.2 关键算法

#### 算法1：唯一性校验

```
输入：文件内容 content，编辑块列表 blocks
输出：匹配位置列表 spans 或报错

对于每个 block in blocks:
    在 content 中查找所有 block.search 的位置
    如果找到 0 处：报错 "SEARCH not found"
    如果找到 >1 处：报错 "SEARCH not unique (n matches)"
    如果找到 1 处：记录匹配范围 (start, end)
返回所有 spans
```

#### 算法2：无重叠校验

```
输入：匹配位置列表 spans
输出：无 或报错

按 start 位置排序 spans
对于每对相邻的 (prev, curr):
    如果 curr.start < prev.end:
        报错 "Overlapping replacement ranges"
```

#### 算法3：原子化替换

```
输入：文件内容 content，匹配位置列表 spans
输出：新内容 new_content

按 start 位置倒序排序 spans（从后往前）
对于每个 span in spans:
    new_content = content[:span.start] + span.replace + content[span.end:]
返回 new_content
```

**关键技巧**：倒序替换避免位置漂移

---

## 3. 落地实施

### 3.1 实施方式

推荐通过打包成 **SKILL 规范** + **AGENTS.md规则约束**实现：

- **AGENTS.md规则约束**: 推荐提示词:

```
大文件编辑安全协议（safe-fast-edit）

- 当满足任一条件时，禁止继续逐条 `edit`，必须切换 `safe-fast-edit` SKILL：

  1) 同一文件 `edit` 连续失败 >= 2 次（如 line changed / text not found / 匹配歧义）

  2) 目标文件 > 300 行 且本次修改点 >= 3 处

- 切换后使用批量 `SEARCH/REPLACE` JSON 一次提交，执行相应脚本;

- 每个 `search` 必须带上下文并在全文唯一命中（count==1）；多块禁止重叠。

- 先 `--dry-run` 预检，通过后再正式写入；必要时加 `--backup`。

- 失败时先 `read` 目标文件重新取锚点，禁止盲改与并发多次 edit。
```

- **SKILL 规范**：定义触发条件、使用流程、约束条件（AGENTS.md 或独立 SKILL 文件）
	- Python 脚本：实现核心逻辑（`smart_patch.py`）, 打包在SKILL包内
- **集成方式**：Agent 检测到触发条件时，自动调用SKILL, 然后使用脚本执行

### 3.2 文件组织结构

```
opencode-project/
├── .opencode/
│   └── skills/
│       └── safe-fast-edit/
│           ├── SKILL.md           # 使用规范和触发条件
│           └── scripts/
│           |   └── smart_patch.py # 核心实现
|           └── examples/ edits.sample.json # 替换内容存放点
└── AGENTS.md                      # 可选：在手册中定义使用规则
```

### 3.3 完整代码实现

以下是 `smart_patch.py` 的完整代码（可直接复制使用）：

```python
#!/usr/bin/env python3
"""
Safe batch SEARCH/REPLACE editor - 大文件编辑安全协议核心实现

设计原则：
1. 唯一锚点：SEARCH 块必须全文唯一匹配
2. 批量原子提交：所有修改在内存中完成，验证通过后再写盘
3. 预校验机制：写入前检查命中唯一性和块间冲突
"""

from __future__ import annotations

import argparse
import json
import sys
from dataclasses import dataclass
from pathlib import Path
from typing import List


@dataclass
class EditBlock:
    """编辑块数据结构"""
    index: int       # 块序号（用于错误提示）
    search: str      # 要搜索的文本（必须唯一）
    replace: str     # 替换后的文本


@dataclass
class MatchSpan:
    """匹配位置数据结构"""
    block: EditBlock # 对应的编辑块
    start: int       # 匹配起始位置
    end: int         # 匹配结束位置


def parse_args() -> argparse.Namespace:
    """解析命令行参数"""
    parser = argparse.ArgumentParser(description="Safe batch SEARCH/REPLACE editor")
    parser.add_argument("file_path", help="Target file path")
    
    # 编辑数据源（二选一）
    source = parser.add_mutually_exclusive_group(required=True)
    source.add_argument("--edits-json", help="Inline edits JSON string")
    source.add_argument("--edits-file", help="Path to edits JSON file")
    
    parser.add_argument(
        "--dry-run",
        action="store_true",
        help="Validate only, do not write file",
    )
    parser.add_argument(
        "--backup",
        action="store_true",
        help="Create .bak backup before writing",
    )
    return parser.parse_args()


def load_edits(args: argparse.Namespace) -> List[EditBlock]:
    """加载并解析编辑数据"""
    raw: str
    if args.edits_json is not None:
        raw = args.edits_json
    else:
        edits_file = Path(args.edits_file)
        if not edits_file.exists():
            raise ValueError(f"Edits file does not exist: {edits_file}")
        raw = edits_file.read_text(encoding="utf-8")
    
    try:
        payload = json.loads(raw)
    except json.JSONDecodeError as exc:
        raise ValueError(f"Failed to parse JSON: {exc}") from exc
    
    if not isinstance(payload, list) or not payload:
        raise ValueError("Edits must be a non-empty JSON array")
    
    blocks: List[EditBlock] = []
    for i, item in enumerate(payload, start=1):
        if not isinstance(item, dict):
            raise ValueError(f"Block {i} must be an object")
        if "search" not in item or "replace" not in item:
            raise ValueError(f"Block {i} missing search or replace")
        search = item["search"]
        replace = item["replace"]
        if not isinstance(search, str) or not isinstance(replace, str):
            raise ValueError(f"Block {i} search/replace must be strings")
        if search == "":
            raise ValueError(f"Block {i} search cannot be empty")
        blocks.append(EditBlock(index=i, search=search, replace=replace))
    return blocks


def find_all_positions(content: str, needle: str) -> List[int]:
    """在内容中查找所有匹配位置"""
    positions: List[int] = []
    start = 0
    while True:
        idx = content.find(needle, start)
        if idx == -1:
            break
        positions.append(idx)
        start = idx + 1
    return positions


def validate_uniqueness(content: str, blocks: List[EditBlock]) -> List[MatchSpan]:
    """验证每个编辑块的 SEARCH 是否全文唯一"""
    spans: List[MatchSpan] = []
    for block in blocks:
        positions = find_all_positions(content, block.search)
        count = len(positions)
        if count == 0:
            raise ValueError(
                f"SEARCH not found (Block {block.index}): {block.search[:80]!r}"
            )
        if count > 1:
            raise ValueError(
                f"SEARCH is not unique (Block {block.index}, {count} matches): {block.search[:80]!r}"
            )
        start = positions[0]
        spans.append(
            MatchSpan(
                block=block,
                start=start,
                end=start + len(block.search),
            )
        )
    return spans


def validate_no_overlap(spans: List[MatchSpan]) -> None:
    """验证多个替换块之间是否有重叠"""
    ordered = sorted(spans, key=lambda s: s.start)
    for prev, curr in zip(ordered, ordered[1:]):
        if curr.start < prev.end:
            raise ValueError(
                "Overlapping replacement ranges detected: "
                f"Block {prev.block.index} ({prev.start}-{prev.end}) and "
                f"Block {curr.block.index} ({curr.start}-{curr.end})"
            )


def build_new_content(content: str, spans: List[MatchSpan]) -> str:
    """在内存中构建新内容（倒序替换避免位置漂移）"""
    new_content = content
    for span in sorted(spans, key=lambda s: s.start, reverse=True):
        new_content = (
            new_content[: span.start] + span.block.replace + new_content[span.end :]
        )
    return new_content


def apply_smart_patch(
    file_path: Path,
    blocks: List[EditBlock],
    dry_run: bool,
    backup: bool,
) -> int:
    """应用智能补丁"""
    if not file_path.exists():
        print(f"ERROR: file does not exist: {file_path}")
        return 1
    if not file_path.is_file():
        print(f"ERROR: target is not a file: {file_path}")
        return 1
    
    # 保留原换行符（LF/CRLF）
    content = file_path.read_text(encoding="utf-8", newline="")
    
    try:
        spans = validate_uniqueness(content, blocks)
        validate_no_overlap(spans)
    except ValueError as exc:
        print(f"ERROR: {exc}")
        return 1
    
    new_content = build_new_content(content, spans)
    print(f"OK: preflight passed for {len(blocks)} blocks")
    
    if dry_run:
        changed = "yes" if new_content != content else "no"
        print(f"OK: dry-run finished, content changed = {changed}")
        return 0
    
    if backup:
        backup_path = file_path.with_suffix(file_path.suffix + ".bak")
        backup_path.write_text(content, encoding="utf-8", newline="")
        print(f"OK: backup created: {backup_path}")
    
    try:
        file_path.write_text(new_content, encoding="utf-8", newline="")
    except OSError as exc:
        print(f"ERROR: failed to write file: {exc}")
        return 1
    
    print(f"OK: wrote {len(blocks)} changes to {file_path.name}")
    return 0


def main() -> int:
    """主函数"""
    args = parse_args()
    try:
        blocks = load_edits(args)
    except ValueError as exc:
        print(f"ERROR: invalid arguments: {exc}")
        return 1
    
    target = Path(args.file_path)
    return apply_smart_patch(
        file_path=target,
        blocks=blocks,
        dry_run=args.dry_run,
        backup=args.backup,
    )


if __name__ == "__main__":
    sys.exit(main())
```

### 3.4 使用方式

**基本用法**：
```bash
python .opencode/skills/safe-fast-edit/scripts/smart_patch.py "target.md" \
  --edits-json '[{"search":"旧文本\n上下文","replace":"新文本\n上下文"}]'
```

**预演模式（不写入）**：
```bash
python .opencode/skills/safe-fast-edit/scripts/smart_patch.py "target.md" \
  --edits-file "edits.json" \
  --dry-run
```

**带备份**：
```bash
python .opencode/skills/safe-fast-edit/scripts/smart_patch.py "target.md" \
  --edits-file "edits.json" \
  --backup
```

**JSON 格式示例**（`edits.json`）：
```json
[
  {
    "search": "旧文本第一行\n旧文本第二行",
    "replace": "新文本第一行\n新文本第二行"
  },
  {
    "search": "另一段旧文本",
    "replace": "另一段新文本"
  }
]
```

---

## 4. 其他补充

### 4.1 推荐使用方式

**建议：先照搬，理解后自主优化**

1. **第一阶段（照搬）**：
   - 复制 `smart_patch.py` 到项目 `.opencode/skills/safe-fast-edit/scripts/`
   - 在 AGENTS.md 中添加触发条件和切换规则
   - 运行验证，确保基本功能正常

2. **第二阶段（理解）**：
   - 阅读代码中的注释，理解三个核心算法
   - 尝试修改一些边界情况的处理逻辑

3. **第三阶段（优化）**：
   - 根据项目特点调整触发阈值（如修改默认的 300 行）
   - 添加项目特定的校验逻辑
   - 集成到 Agent 工作流中自动触发

### 4.2 关键坑点

#### 坑点1：换行符不匹配（Windows 最常见）

**问题**：Windows 文件使用 `CRLF` (`\r\n`)，但编辑时只输入 `LF` (`\n`)

**解决**：
- 使用 `read` 工具读取文件，复制原文中的换行符
- Python 脚本使用 `newline=""` 保留原换行符
- 如果无法确定，先创建 `test.txt` 测试 SEARCH 是否匹配

#### 坑点2：SEARCH 块不够唯一

**问题**：使用简短文本作为 SEARCH，结果在文件中出现多次

**解决**：
- SEARCH 必须包含至少 2 行上下文
- 确保全文只有一处匹配
- 使用 `--dry-run` 预演，会提示 "n matches" 错误

#### 坑点3：块间重叠

**问题**：两个替换范围有交集，导致冲突

**解决**：
- 合并为一个大的替换块
- 或调整锚点位置，确保不重叠
- 错误信息会提示具体哪两个块冲突

#### 坑点4：JSON 转义问题

**问题**：命令行传递 JSON 时，特殊字符（引号、换行）导致解析失败

**解决**：
- 复杂编辑使用 `--edits-file` 从文件读取
- 简单编辑确保正确转义：`"` → `\"`，换行 → `\n`

### 4.3 失败重试流程

当遇到错误时，按以下步骤排查：

```
1. 读取原始文件 → 确认 SEARCH 文本确实存在
2. 检查唯一性 → 全文搜索，看是否有重复
3. 添加上下文 → 扩展 SEARCH 包含更多唯一特征行
4. 预演验证 → 使用 --dry-run 测试
5. 正式执行 → 去除 --dry-run 运行
```

### 4.4 验收标准

一个成功实施的 safe-fast-edit 应该满足：

**功能验收**：
- [ ] 能够成功修改 300+ 行文件的多处内容
- [ ] edit 失败 ≥2 次后自动切换到此方案
- [ ] 预校验失败时原文件保持不变
- [ ] 支持 `--dry-run` 预演模式
- [ ] 支持 `--backup` 创建备份

**质量验收**：
- [ ] SEARCH 块在全文唯一（命中数 = 1）
- [ ] 修改块之间无重叠
- [ ] 保留原文件换行符（LF/CRLF）
- [ ] 错误信息清晰，指出具体问题和位置

**边界情况**：
- [ ] 文件不存在时报错
- [ ] SEARCH 未找到时报错
- [ ] SEARCH 不唯一时报错（提示匹配数）
- [ ] 块重叠时报错（指出冲突块）
- [ ] 写入失败时保留原文件

### 4.5 扩展阅读

**相关概念**：
- **原子化操作**：要么全成功，要么全失败，没有中间状态
- **乐观锁**：先假设能成功，提交前再校验，失败则回滚
- **倒序替换**：从后往前修改，避免前面修改影响后面位置

**类似方案**：
- `sed` 的批量替换（但无唯一性校验）
- IDE 的重构功能（基于 AST，非文本匹配）
- Git 的 `apply` 命令（冲突检测机制类似）

**关键词搜索**（如需了解更多）：
- "atomic file write python"
- "search replace unique anchor text"
- "batch edit conflict detection"

---

*最后更新：2026-02-23*
