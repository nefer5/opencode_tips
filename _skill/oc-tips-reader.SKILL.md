---
name: oc-tips-reader
description: 读取 opencode_tips 仓库的规范指南
trigger: [读取OC技巧, OC tips, 仓库 tips, opencode_tips]
---

# OC Tips Reader SKILL

本SKILL用于读取和解析 opencode_tips 仓库。

## 仓库地址

使用 webfetch 读取 GitHub Raw URL：
- 格式：`https://raw.githubusercontent.com/{用户}/{仓库}/main/{文件路径}`
- 示例：`https://raw.githubusercontent.com/nefer/opencode_tips/main/_index/_registry.yaml`
- 无需任何环境配置，不搞 git clone

## 仓库结构

```
opencode_tips/
├── README.md              # 人类友好入口
├── ARCHITECTURE.md       # 架构设计文档
├── _index/
│   ├── index.md          # Agent主索引
│   └── _registry.yaml    # 内容注册表（关键！）
├── tips/                 # 技巧类
├── prompts/              # 提示词类
├── notes/                # 笔记类
└── _skill/
    └── oc-tips-reader.SKILL.md  # 本文件
```

## 读取流程

1. **读取注册表**：
   - 用 webfetch 读取 `{仓库RawURL}/_index/_registry.yaml`
   - 获取 `last_published` 字段

2. **对比缓存**：
   - 缓存文件路径：`{本SKILL目录}/cache.json`
   - 读取本地缓存的 `last_read` 字段
   - 如果 `last_published` <= `last_read`，则无新内容，跳过

3. **获取内容列表**：
   - 解析 `_registry.yaml` 的 `contents` 数组
   - 每个条目包含：id, type, title, file, published, status, tags

4. **读取详细内容**：
   - 用 webfetch 读取 `{仓库RawURL}/{file}`
   - 只读取 `status: published` 的内容

5. **按类型处理**：
   - `tip` → 当作技巧讲解给用户
   - `prompt` → 当作模板提供使用
   - `note` → 当作参考笔记
   - `skill` → 完整SKILL格式，可加载

6. **检查系统台账**：
   - 读取本地 `SYSTEM_REGISTRY.md`（系统能力清单）
   - 对比新技巧与已具备能力，标记 `已掌握` / `待学习`
   - 示例：如果已会 GTD，则标记为复习；如果是新能力，则标记为待实践

7. **与用户讨论**：
   - 介绍新内容
   - 询问用户哪些想实践
   - 根据系统台账给出建议（已掌握的技巧可跳过）

8. **更新缓存**：
   - 读取完成后更新 `{本SKILL目录}/cache.json` 的 `last_read` 为当前时间

## 内容格式

### 技巧文件 (tips/*.md)

```yaml
---
title: 技巧标题
type: tip
published: 2026-02-20
status: published
tags: [tag1, tag2]
---

# 技巧标题

## 内容...
```

### 提示词文件 (prompts/*.md)

```yaml
---
title: 提示词名称
type: prompt
published: 2026-02-20
status: published
tags: [tag1]
---

# 提示词名称

## 使用场景
...

## 提示词模板
\`\`\`
...
\`\`\`
```

## 示例对话

用户：帮我看看最近有什么新技巧

Agent：
> 让我检查一下 opencode_tips 仓库的更新...
> 
> 上次读取时间：2026-02-20
> 最新发布时间：2026-02-23
> 
> 有新内容！让我读取一下：
> 
> **tip-001: 如何用GTD管理任务**
> - 类型：tip
> - 标签：gtd, 任务管理
> - 发布时间：2026-02-23
> 
> 要我帮你把这个技巧应用到你当前的任务管理中吗？

## 系统台账检查

每次读取新内容后，请：

1. **读取 SYSTEM_REGISTRY.md**：
   - 路径：`{opencode配置目录}/system/SYSTEM_REGISTRY.md`
   - 或用户指定的系统能力记录文件

2. **能力比对**：
   - 检查新技巧是否已在系统台账中
   - 如果已掌握 → 标记为"复习"
   - 如果未掌握 → 标记为"待学习"

3. **智能推荐**：
   - 优先推荐与当前技能栈互补的内容
   - 避免重复推荐已掌握的技巧

## 注意事项

- 只读取 `status: published` 的内容
- 跳过 `status: draft`（草稿）和 `status: deprecated`（过时）
- 如果没有新内容，告诉用户"没有新内容"
- 始终尊重用户的讨论需求，解释内容而不是照搬
- **缓存统一放在本SKILL目录内**，不分散到系统其他位置
