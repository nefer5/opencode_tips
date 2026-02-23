---
name: oc-tips-writer
description: 发布方快速创建和更新 opencode_tips 仓库内容的工具
trigger: [发布新技巧, 更新仓库, 添加tips, oc-tips发布]
---

# OC Tips Writer SKILL

本SKILL用于发布方（我）快速创建和更新 opencode_tips 仓库内容。

## 使用场景

当我要分享一个新的 OpenCode 技巧、提示词或笔记时，使用此SKILL快速创建规范格式的内容。

## 发布流程

### 方式一：对话式创建（推荐）

直接告诉我你要发布什么：

```
用户：我要发布一个新技巧，关于如何用skill快速生成文档

Agent：
好的，我来帮你创建这个技巧。请告诉我：

1. **标题**：如何用 skill 快速生成文档
2. **类型**：tip（技巧）
3. **标签**：[skill, 文档生成, 效率]
4. **内容要点**：
   - 步骤1：...
   - 步骤2：...
   - 注意事项：...

请补充完整内容，或我根据你的描述生成初稿？
```

### 方式二：批量导入

如果你有现成的 Markdown 文件，可以直接导入并转换格式。

## 自动处理事项

当你确认发布后，我会自动：

1. **生成文件**：按规范格式创建 `.md` 文件
2. **分配ID**：自动分配递增的ID（如 `tip-003`, `prompt-002`）
3. **更新时间**：自动设置 `published` 为当前日期
4. **注册内容**：自动更新 `_registry.yaml`
5. **格式检查**：确保符合仓库规范

## 文件命名规范

| 类型 | 命名格式 | 示例 |
|------|----------|------|
| tip | `tips/XXX-标题.md` | `tips/001-gtd-usage.md` |
| prompt | `prompts/XXX-标题.md` | `prompts/001-code-review.md` |
| note | `notes/XXX-标题.md` | `notes/001-oc-workflow.md` |

## 生成的文件格式

### Tip 示例

```markdown
---
title: 如何用GTD管理任务
type: tip
published: 2026-02-23
status: published
tags: [gtd, 任务管理]
---

# 如何用GTD管理任务

## 核心思想

GTD（Getting Things Done）是一种任务管理方法...

## 具体步骤

1. **收集**：把所有任务放进 Inbox
2. **整理**：分类处理
3. **执行**：按情境选择任务

## 在 OpenCode 中的应用

- 使用 skill-gtd 插件
- 设置每周回顾提醒

## 参考

- 相关文件：[GTD workflow](../notes/001-gtd-workflow.md)
```

### Prompt 示例

```markdown
---
title: 代码审查提示词
type: prompt
published: 2026-02-23
status: published
tags: [代码审查, prompt]
---

# 代码审查提示词

## 使用场景

当你需要审查一段代码时...

## 提示词模板

```
请审查以下代码：

```代码类型
{代码内容}
```

请检查：
1. 潜在bug
2. 性能问题
3. 可读性改进建议
4. 最佳实践
```

## 示例对话

用户：帮我审查这段 Python 函数

Agent：...
```

## 快速命令

| 命令 | 作用 |
|------|------|
| `发布新技巧 [标题]` | 创建 tip 文件 |
| `发布提示词 [标题]` | 创建 prompt 文件 |
| `发布笔记 [标题]` | 创建 note 文件 |
| `查看待发布` | 列出草稿状态的内容 |
| `标记发布 [ID]` | 将草稿标记为已发布 |

## 发布检查清单

创建内容后，请检查：

- [ ] 标题是否清晰明确
- [ ] 标签是否准确（2-5个）
- [ ] 内容是否完整（步骤/示例/说明）
- [ ] 是否有相关链接
- [ ] 是否标记了 `status: published`
- [ ] `_registry.yaml` 是否已更新

## 注意事项

1. **ID 递增**：自动分配，不要手动修改已有ID
2. **文件名**：使用小写、连字符 `-` 连接
3. **标签规范**：统一使用中文或英文，不要混用
4. **链接格式**：相对路径 `./xxx.md` 或 `../xxx/xxx.md`
5. **草稿管理**：不确定的内容先用 `status: draft`
6. **Git 提交**：内容创建后记得 `git add . && git commit -m "add: xxx"`

## 示例对话

### 场景1：快速发布技巧

用户：我要发布一个新技巧，关于"如何用webfetch抓取网页"

Agent：
好的，我来帮你创建这个技巧。

**自动生成信息**：
- 类型：tip
- ID：tip-005（下一个可用ID）
- 文件名：`tips/005-webfetch-guide.md`
- 发布时间：2026-02-23

请补充以下内容：
1. 这个技巧解决什么问题？
2. 具体步骤是什么？
3. 有什么注意事项？
4. 可以举个实际例子吗？

或者直接说"你帮我写"，我根据经验生成初稿？

---

用户：你帮我写，重点是如何处理网页中的中文编码

Agent：
好的，我为你生成内容：

```markdown
[生成内容展示...]
```

内容已生成，请确认：
- [ ] 标题是否准确？
- [ ] 步骤是否完整？
- [ ] 示例是否清晰？

确认后我将：
1. 保存到 `tips/005-webfetch-guide.md`
2. 更新 `_registry.yaml`
3. 提示你 git commit

---

用户：确认

Agent：
✅ 已完成！

创建文件：`tips/005-webfetch-guide.md`
更新注册表：`_registry.yaml`（新增 tip-005）

请执行：
```bash
git add .
git commit -m "add: tip-005 webfetch中文编码处理"
git push
```

### 场景2：从已有内容导入

用户：我本地有个 `my-tip.md`，帮我导入到仓库

Agent：
好的，我来读取并转换格式。

**检测到的内容**：
- 原标题：XXX
- 建议类型：tip
- 建议ID：tip-006

是否需要：
1. 保留原文内容
2. 添加 YAML frontmatter
3. 转换为标准格式

---

用户：可以，顺便帮我加个标签 [web, api]

Agent：
✅ 转换完成！

已保存到：`tips/006-xxx.md`
标签：[web, api]

请检查内容，确认后提交到 git。

## 工具函数（供Agent使用）

### 1. 获取下一个ID

读取 `_registry.yaml`，按类型找出最大ID，+1

### 2. 验证文件名

检查是否符合 `XXX-标题.md` 格式，标题是否合法

### 3. 生成Frontmatter

```python
def generate_frontmatter(title, type, tags, published_date=None):
    if not published_date:
        published_date = today()
    return f"""---
title: {title}
type: {type}
published: {published_date}
status: draft  # 默认草稿，确认后改 published
tags: {tags}
---
"""
```

### 4. 更新Registry

```python
def add_to_registry(content_id, type, title, file, tags):
    # 读取现有 registry
    # 追加新条目
    # 按ID排序
    # 写回文件
```

## 错误处理

| 错误场景 | 处理方式 |
|----------|----------|
| ID 冲突 | 提示用户，自动跳过到下一个可用ID |
| 文件名已存在 | 提示更换标题或覆盖 |
| 标签格式错误 | 自动转换为标准格式 |
| Registry 格式错误 | 备份并重建 |

## 最佳实践

1. **批量创建**：一次不要创建超过3个内容，保证质量
2. **及时提交**：创建后立即 git commit，避免丢失
3. **版本控制**：重要修改先创建分支
4. **定期整理**：每月检查一次 `draft` 状态的内容
5. **链接维护**：移动文件时更新所有引用链接

---

*本SKILL帮助发布方高效管理 opencode_tips 仓库内容*
