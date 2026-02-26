---
id: note-002
type: note
title: "OpenSpec 工作流完整指南：从想法到实现的标准化路径"
published: "2026-02-27"
status: draft
tags: [openspec, workflow, ulw, 项目管理, 结构化开发, 变更管理]
---

# OpenSpec 工作流完整指南：从想法到实现的标准化路径

## 1. 背景：要解决的问题

### 1.1 问题场景

在 OpenCode 中进行复杂功能开发时，经常面临以下困境：

| 问题类型 | 典型表现 | 发生场景 |
|---------|---------|---------|
| **需求不明确** | 做到一半发现理解错了，需要推倒重来 | 用户说"做个XX功能"，没有详细说明 |
| **设计决策混乱** | 临时决定，后期发现方案有问题 | 边做边想，缺乏前置设计 |
| **实现过程缺乏追踪** | 不知道做到哪一步，任务遗漏 | 多步骤任务没有系统管理 |
| **代码与需求脱节** | 实现的功能不是用户想要的 | 没有明确的规格文档对照 |
| **经验无法复用** | 每次遇到类似问题都要重新思考 | 缺乏结构化的方案沉淀 |

### 1.2 问题根源

为什么会出现这些问题？

1. **缺乏结构化流程**：传统开发模式从需求直接跳到代码，缺少中间的设计和规格阶段
2. **变更不可追溯**：需求变化、设计调整没有记录，导致后期难以理解和维护
3. **上下文丢失**：实现过程中的决策理由、取舍考量没有被记录下来
4. **协作困难**：多人协作时，每个人的理解可能不一致，缺少统一的文档基准

### 1.3 触发条件

什么情况下应该考虑使用 OpenSpec 工作流？

| 条件 | 阈值 | 说明 |
|------|------|------|
| 任务复杂度 | 涉及 3+ 个模块或文件 | 简单单文件修改不需要 |
| 不确定性 | 需求或方案有多个可能 | 需要探索澄清 |
| 可复用性 | 类似场景可能重复发生 | 值得沉淀为规范 |
| 协作需求 | 需要与他人讨论方案 | 需要文档作为讨论基础 |
| 追踪需求 | 需要记录决策过程 | 后期审计或学习 |

---

## 2. 解决思路：OpenSpec 核心理念

### 2.1 核心设计理念

OpenSpec 提供了一套**结构化的变更管理流程**，将开发工作划分为四个明确的阶段：

```
┌─────────────────────────────────────────────────────────────┐
│                      OpenSpec 工作流                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   想法/需求                                                  │
│      │                                                       │
│      ▼                                                       │
│   ┌──────────────┐    澄清需求、探索方案                      │
│   │   Explore    │    识别隐藏意图、评估可行性                 │
│   └──────────────┘                                          │
│      │                                                       │
│      ▼                                                       │
│   ┌──────────────┐    生成完整设计文档                        │
│   │   Propose    │    proposal + design + specs + tasks       │
│   └──────────────┘                                          │
│      │                                                       │
│      ▼                                                       │
│   ┌──────────────┐    按任务清单执行                          │
│   │    Apply     │    逐步实现，自动跟踪进度                   │
│   └──────────────┘                                          │
│      │                                                       │
│      ▼                                                       │
│   ┌──────────────┐    完成归档，变更历史永久保存               │
│   │   Archive    │    支持后期追溯和学习                       │
│   └──────────────┘                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**关键原则**：
- **先设计后实现**：不允许跳过设计阶段直接写代码
- **文档即代码**：每个阶段都有明确的产出物（artifacts）
- **可追溯**：所有决策、变更都有记录
- **可复现**：任何人都可以按照文档重建相同的方案

### 2.2 关键逻辑：四阶段详解

#### Stage 1: Explore（探索）

**目标**：澄清需求，探索可能的方案

**核心活动**：
- 与用户讨论，明确真实需求
- 探索现有代码库，了解上下文
- 识别隐藏意图和潜在风险
- 比较不同方案的优劣

**产出**：清晰的任务描述、功能范围、约束条件

#### Stage 2: Propose（提案）

**目标**：生成完整的设计文档

**核心活动**：
- 撰写 `proposal.md`：变更动机和范围
- 撰写 `design.md`：技术架构和决策
- 撰写 `specs/<capability>/spec.md`：详细功能规格
- 撰写 `tasks.md`：实现任务清单

**产出**：完整的 artifacts，作为后续实现的蓝图

#### Stage 3: Apply（应用）

**目标**：按照设计文档执行实现

**核心活动**：
- 读取 design 和 tasks 获取上下文
- 按任务清单逐项实现
- 每完成一项标记为 done
- 遇到问题时暂停，更新设计或询问用户

**产出**：实际的可工作的代码

#### Stage 4: Archive（归档）

**目标**：完成变更，保存历史记录

**核心活动**：
- 验证所有任务完成
- 将 artifacts 归档到 `openspec/archive/`
- 清理临时文件

**产出**：可追溯的变更历史

---

## 3. 落地实施

### 3.1 实施方式

通过 **OpenSpec CLI + OpenCode 插件** 实现。

**官方资源**：
- OpenSpec CLI: https://openspec.dev
- OpenCode Plugin: https://github.com/opencode-ai/opencode-plugin-openspec

### 3.2 安装步骤

#### 步骤1：安装 OpenSpec CLI（全局安装）

```bash
npm install -g @openspec/cli
```

验证安装：
```bash
openspec --version
```

#### 步骤2：配置 OpenCode 插件

编辑 OpenCode 配置文件（`~/.config/opencode/opencode.json`），添加插件声明：

```json
{
  "models": {
    // ... 你的模型配置
  },
  "plugins": [
    {
      "name": "opencode-plugin-openspec",
      "config": {}
    }
  ]
}
```

#### 步骤3：初始化 OpenSpec（项目级）

在你的项目根目录下执行：

```bash
openspec init
```


这会在当前目录创建 `openspec/` 目录结构，这是 OpenSpec 工作的基础。

**⚠️ 注意**：必须在项目文件夹下执行一次，否则 OpenCode 插件无法识别项目上下文。

#### 步骤4：重启 OpenCode

完全退出并重新启动 OpenCode，让插件生效。

#### 步骤5：验证安装

在 OpenCode 中输入以下命令，验证插件已加载：

```
/opsx-explore
```

如果看到帮助信息，说明安装成功。

#### 步骤3：重启 OpenCode

完全退出并重新启动 OpenCode，让插件生效。

#### 步骤4：验证安装

在 OpenCode 中输入以下命令，验证插件已加载：

```
/opsx-explore
```

如果看到帮助信息，说明安装成功。

### 3.3 核心命令速查

| 命令 | 功能 | 示例 |
|------|------|------|
| `/opsx-explore [主题]` | 进入探索模式 | `/opsx-explore add-auth` |
| `/opsx-propose <name>` | 创建变更提案 | `/opsx-propose file-batch-renamer` |
| `/opsx-apply [name]` | 执行任务实现 | `/opsx-apply file-batch-renamer` |
| `/opsx-archive [name]` | 归档完成变更 | `/opsx-archive file-batch-renamer` |
| `openspec list` | 查看所有变更 | - |
| `openspec status --change <name>` | 查看变更状态 | - |

### 3.4 完整使用示例

#### 场景：开发一个新 Skill

**步骤1：启动满血模式（重要！）**

在执行开始前，加上 `ulw` 魔法提示词：

```
/opsx-propose ulw 帮我开发一个文件批量重命名 Skill，支持多种规则组合
```

> **关键提示**：`ulw` 会激活 ultrawork-mode，最大化并行 agent 调用，显著缩短复杂任务的执行时间。对于 OpenSpec 工作流这种多步骤、需要并行探索的任务，强烈推荐使用。

**步骤2：探索阶段（Explore）**

```
/opsx-explore 这里可以blabla讨论一堆
```

在此阶段：
- 澄清需求：需要什么功能？8种规则？目录名前缀？
- 探索代码库：查看现有 Skill 结构
- 确定功能范围

**步骤3：提案阶段（Propose）**

```
/opsx-propose 这里就是定方案了, 出设计 file-batch-renamer
```

OpenCode 会自动生成：
- `openspec/changes/file-batch-renamer/proposal.md`
- `openspec/changes/file-batch-renamer/design.md`
- `openspec/changes/file-batch-renamer/specs/batch-rename/spec.md`
- `openspec/changes/file-batch-renamer/tasks.md`

**步骤4：应用阶段（Apply）**

```
/opsx-apply ulw file-batch-renamer
```

OpenCode 会：
1. 读取 design.md 和 tasks.md
2. 按任务清单逐项实现
3. 每完成一项更新 tasks.md
4. 直到所有任务完成

**步骤5：归档阶段（Archive）**

```
/opsx-archive file-batch-renamer
```

变更历史会被保存到 `openspec/archive/`。

### 3.5 实战案例简要总结

本次 `file-batch-renamer` Skill 开发完整路径：

| 阶段 | 耗时 | 产出 | 关键活动 |
|------|------|------|---------|
| **Explore** | ~10分钟 | 功能范围确定 | 需求对齐：8种规则 + 目录名前缀（dirA_1, dirA_2...） |
| **Propose** | ~15分钟 | 4个 artifacts | 生成 proposal/design/specs/tasks，共 30 项任务 |
| **Apply** | ~30分钟 | 完整实现 | 641行核心代码 + 测试套件 + 使用文档 |
| **Archive** | ~2分钟 | 归档完成 | 变更历史永久保存 |

**核心代码结构**：
```
opencode/skills/file-batch-renamer/
├── SKILL.md                      # Skill 入口（触发词定义）
├── scripts/
│   ├── renamer.py               # 641行核心实现
│   └── test_renamer.py          # 测试套件
└── references/
    ├── usage-guide.md           # 详细使用指南
    └── troubleshooting.md       # 故障排查
```

**功能特性**：
- 8种重命名规则（前缀/后缀、序号、正则、目录名、日期、大小写、分隔符）
- 规则组合：多个规则按顺序应用
- 预览模式：执行前展示变更清单
- 撤销功能：一键恢复，自动保存历史
- 安全保护：冲突检测、非法字符检查

---

## 4. 其他补充

### 4.1 推荐使用方式

**适合照搬的场景**：
- 复杂功能开发（>3个文件/模块）
- 需要记录决策过程的项目
- 多人协作的变更管理

**建议改造的场景**：
- 简单单文件修改（直接编辑更快）
- 紧急修复（可简化流程，但建议事后补文档）

### 4.2 关键坑点

#### 坑点1：忘记使用 `ulw` 满血模式

**问题**：OpenSpec 流程涉及多步骤、并行探索、agent 协调，普通模式下执行缓慢。

**解决**：任务开始前务必加上 `ulw` 提示词：
```
/opsx-propose ulw 你想做个什么东西, 随便说, my-change
```

满血模式会自动：
- 并行启动多个 explore/librarian agent
- 最大化工具调用效率
- 减少等待时间 50%+

#### 坑点2：Explore 阶段不充分

**问题**：急于进入 Propose，导致需求理解偏差，后期返工。

**解决**：Explore 阶段多花 10-15 分钟，确保：
- 功能范围明确
- 约束条件清晰
- 技术方案可行

#### 坑点3：跳过设计文档直接 Apply

**问题**：不看 design.md 直接写代码，实现与设计不符。

**解决**：Apply 阶段必须先读取上下文文件（design + tasks），按设计执行。

#### 坑点4：任务清单更新不及时

**问题**：完成任务后忘记标记，导致重复工作或遗漏。

**解决**：每完成一项立即更新 tasks.md，`- [ ]` → `- [x]`。

#### 坑点5：归档后找不到历史记录

**问题**：变更归档后不知道去哪里找。

**解决**：归档后的变更保存在 `openspec/archive/<change-name>/`，包含所有 artifacts。

#### 坑点6：有时会卡住卡很久

**问题**：感觉哪里有点bug

**解决**：esc中断下, 这个openspec在后台会复苏继续. 万一它不继续, 你说下继续就行.  

### 4.3 验收标准

- [ ] 能成功安装 OpenSpec CLI 和 OpenCode 插件
- [ ] 能使用 `/opsx-explore` 进入探索模式
- [ ] 能使用 `/opsx-propose` 创建变更提案
- [ ] 能使用 `/opsx-apply` 执行任务
- [ ] 能使用 `/opsx-archive` 归档变更
- [ ] 理解 `ulw` 魔法提示词的作用并使用
- [ ] 能在复杂任务中应用四阶段工作流

### 4.4 扩展阅读

**相关概念**：
- [OpenSpec 官方文档](https://openspec.dev)
- [OpenCode Plugin GitHub](https://github.com/opencode-ai/opencode-plugin-openspec)
- [Ultrawork Mode 说明](https://docs.opencode.ai/ultrawork)

**类似方案**：
- Git Flow：分支管理模型
- GitHub Flow：PR 工作流
- RFC 流程：设计文档先行

**关键词搜索**：
- "OpenSpec schema design"
- "structured change management"
- "spec-driven development"

---

**重点提醒**：对于 OpenSpec 这类复杂工作流任务，执行开始时务必加上 `ulw` 魔法提示词，满血启动 ultrawork-mode，最大化并行 agent 调用，将整个流程的执行时间压缩到最短。

---

*最后更新：2026-02-27*  
*版本：v1.0 - 基于 file-batch-renamer 实战案例总结*
