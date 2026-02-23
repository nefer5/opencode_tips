---
title: 如何使用 System Registry 管理系统台账
type: tip
published: 2026-02-23
status: published
tags: [system-registry, 系统台账, 能力管理, opencode]
---

# 如何使用 System Registry 管理系统台账

## 这是什么？

System Registry 是一个用于管理 OpenCode 系统台账的 Skill，帮助你记录能力清单、改造日志和系统状态。

## 核心功能

1. **读取系统状态** - 查看当前系统能力和最近变更
2. **记录变更日志** - 自动追加改造历史
3. **管理功能清单** - 维护系统能力索引

## 快速使用

### 1. 查看系统状态

```bash
python .opencode/skills/system-registry/scripts/system_registry.py --read
```

输出示例：
```
[STATE]
- 最后更新：2026-02-23
- 最近改造模块：opencode-tips
- 本次改造类型：capability
- 当前状态：beta

[LATEST_CHANGES]
- 2026-02-23 17:22 | type=capability | module=opencode-tips | ...
```

### 2. 记录新变更

```bash
python .opencode/skills/system-registry/scripts/system_registry.py --log \
  --type "capability" \
  --module "你的模块名" \
  --change "变更描述" \
  --entry "入口文件路径" \
  --impact "影响说明" \
  --status "stable"
```

**参数说明**：
- `--type`: 变更类型
  - `capability` - 新能力
  - `stability-fix` - 稳定性修复
  - `feature` - 新功能
  - `bugfix` - Bug修复
  - `refactor` - 重构
- `--module`: 改造模块（如 oc, gtd, mem0）
- `--change`: 变更描述（一句话）
- `--entry`: 入口文件/关键文件路径
- `--impact`: 影响说明（用户能感知的变化）
- `--status`: 状态（stable/beta/experimental）

### 3. 更新能力清单

当有新能力上线时，**手动编辑** `.opencode/system/SYSTEM_REGISTRY.md`：

1. 找到 `## 能力清单` 区域
2. 按格式添加：
   ```markdown
   ### 新能力名称
   - **入口**: `.opencode/skills/xxx/`
   - **功能**: 一句话描述
   - **关键词**: 关键词1/关键词2
   ```

## 文件结构

```
.opencode/system/SYSTEM_REGISTRY.md
├── <!-- AUTO_SYSTEM_STATE_START -->
│   ├── 当前系统状态（自动更新）
│   └── 能力清单（手动维护）
├── <!-- AUTO_SYSTEM_STATE_END -->
├── <!-- AUTO_SYSTEM_CHANGES_START -->
│   └── 改造日志（--log 自动追加）
└── <!-- AUTO_SYSTEM_CHANGES_END -->
└── 手动维护区域（功能入口索引等）
```

## 触发关键词

对 Agent 说这些可以触发 system-registry skill：
- "系统台账"
- "更新能力清单"
- "读取系统状态"
- "记录变更"
- "system registry"

## 最佳实践

1. **及时记录** - 每次系统改造后立即 `--log`
2. **手动维护能力清单** - 新功能上线后手动添加到能力清单
3. **保持历史** - 不要删除改造日志，保持可追溯
4. **定期查看** - 每月查看一次系统状态，了解能力演进

## 注意事项

- ⚠️ **能力清单区域需要手动维护**，不要依赖脚本自动生成
- ⚠️ **改造日志通过 `--log` 自动追加**，不要手动修改标记区域
- ⚠️ **保护好 `<!-- AUTO_*_START/END -->` 标记**，不要删除或修改

## 参考

- 系统台账文件：`.opencode/system/SYSTEM_REGISTRY.md`
- Skill 位置：`.opencode/skills/system-registry/`
- 相关 Skill：`skill-creator`（创建新 skill 时也应记录到台账）
