---
name: bp-skill-authoring
description: 指导编写和改进 Agent Skill 文件（SKILL.md），涵盖 YAML frontmatter、精简写作、渐进式披露、常用模式。当用户要创建新 Skill、改进现有 SKILL.md、或询问 Skill 编写规范时使用。
---

# 编写 Agent Skills

## 语言规范

**Skills 使用中文编写**，包括：
- `description`：中文描述
- 章节标题和说明：中文
- 代码示例和标识符：保持英文

## YAML Frontmatter

```yaml
---
name: processing-pdfs                    # 小写，连字符，最多 64 字符
description: 从 PDF 文件中提取文本...      # 中文，第三人称，做什么 + 何时用，最多 1024 字符
---
```

**name 格式：**
- 推荐动名词：`processing-pdfs`, `analyzing-data`, `testing-code`
- 名词短语也可：`pdf-processing`, `data-analysis`
- ❌ 避免：`helper`, `utils`, `tools`, `anthropic-*`, `claude-*`

**description 规则：**
- ✅ 使用**中文**，第三人称："从 PDF 文件中提取文本"
- ❌ 避免："我可以帮你..."、"你可以用这个来..."
- 包含：做什么 + 何时使用 + 关键词

## 核心原则

### 精简至上

只添加 AI 尚不知道的信息：

```markdown
✅ 好（~50 tokens）：
## 锁
使用 TDMutex，不要用 std::mutex：
tdstore::common::TDMutex mutex_;

❌ 差（~150 tokens）：
## 锁
线程安全在多线程应用中很重要。
TDStore 使用 bthread 协程，所以需要特殊的锁...
```

### 约束条件

- SKILL.md 正文：**< 500 行**
- 引用深度：从 SKILL.md 起**只能一层**
- 文件路径：**只用正斜杠**（`reference/guide.md`）

## 项目 Skills 目录结构

```
skills/
├── <skill-name>/                    # 通用技能
│   └── SKILL.md
└── <module>/                        # 模块专项技能
    └── <skill-name>/
        └── SKILL.md
```

**示例：**
```
skills/
├── writing-skills/                  # 通用：编写 Skills 指南
│   ├── SKILL.md
│   └── reference/
└── backend/                         # 模块：后端
    ├── coding-standards/            # 编码规范
    │   └── SKILL.md
    └── storage-modifications/       # 存储改造规范
        └── SKILL.md
```

**命名规则：**
- 模块名：`backend`, `storage`, `scheduler` 等
- 技能名：`coding-standards`, `storage-modifications`, `testing` 等
- 使用小写 + 连字符

## 渐进式披露

单个 Skill 内部结构：

```
skill-name/
├── SKILL.md              # 概览（触发时加载）
├── reference/
│   ├── topic-a.md        # 按需加载
│   └── topic-b.md
└── scripts/
    └── validate.py       # 执行，不加载到上下文
```

在 SKILL.md 中链接到详情：

```markdown
## 快速开始
[基本用法]

## 进阶
**主题 A**：参见 [reference/topic-a.md](reference/topic-a.md)
**主题 B**：参见 [reference/topic-b.md](reference/topic-b.md)
```

## 常用模式

### 模板模式

```markdown
## 输出格式

始终使用以下结构：
# [标题]
## 摘要
## 发现
## 建议
```

### 示例模式

```markdown
## Commit 消息

**示例 1：**
输入：Added auth
输出：`feat(auth): implement JWT authentication`

**示例 2：**
输入：Fixed date bug
输出：`fix(reports): correct timezone conversion`
```

### 带检查清单的工作流

```markdown
## 迁移工作流

复制并跟踪进度：
- [ ] 步骤 1：备份数据库
- [ ] 步骤 2：运行迁移
- [ ] 步骤 3：验证数据

**步骤 1：备份**
运行：`./scripts/backup.sh`
...
```

### 条件工作流

```markdown
## 选择路径

**创建新的？** → 参见下方"创建"
**编辑现有的？** → 参见下方"编辑"
```

## 自由度

| 自由度 | 适用场景 | 示例 |
|--------|----------|------|
| **高** | 多种有效方案 | 代码审查指南 |
| **中** | 有推荐模式 | 报告模板 |
| **低** | 脆弱/关键操作 | 数据库迁移 |

低自由度 = 精确命令，不允许修改。

## 发布前检查清单

- [ ] `name`/`description` 符合格式要求（小写连字符，第三人称，做什么 + 何时用）
- [ ] 正文 < 500 行，引用一层深
- [ ] 示例具体，术语一致
- [ ] 路径使用正斜杠

## 反模式

| ❌ 避免 | ✅ 改为 |
|--------|---------|
| Windows 路径（`docs\file.md`） | Unix 路径（`docs/file.md`） |
| 多选项（"用 A 或 B 或 C"） | 一个默认 + 逃生口 |
| 时效性内容（"2025年8月前"） | "当前" + "旧模式"分节 |
| 深层嵌套引用（A→B→C） | 所有引用从 SKILL.md 发起 |
| 解释 AI 已知的内容 | 只写项目特定信息 |

## 进阶主题

完成初稿后，阅读 [reference/full-best-practices.md](reference/full-best-practices.md) 了解：
- 评估驱动开发
- 与 AI 迭代优化
- 可执行脚本设计
