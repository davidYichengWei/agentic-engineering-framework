# Agentic Engineering Framework

基于 Skill 的模块化 Agentic Engineering 框架——将工程纪律与 AI 能力系统性结合，覆盖从需求澄清到代码审查的完整研发流程，并通过反馈闭环自动沉淀团队工程经验。

> 关于 Agentic Engineering 方法论的完整推导，参见 [《从第一性原理思考 Agentic Engineering》](./agentic_engineering.md)。本框架是该文章的实践落地。

## 快速开始

### 1. 克隆仓库

```bash
git clone https://github.com/davidYichengWei/agentic-engineering-framework.git
cd agentic-engineering-framework
```

### 2. 复制到 Agent 配置目录

不同 agent 使用不同的配置目录，将三个文件夹复制到对应位置：

```bash
# Claude Code
cp -R agents skills commands ~/.claude/

# Codex CLI
cp -R agents skills commands ~/.codex/

# CodeBuddy
cp -R agents skills commands ~/.codebuddy/

# 其他支持类似 skill/agent 结构的 agent
cp -R agents skills commands <your-agent-config-dir>/
```

### 3. 验证

启动与 agent 的对话，尝试一个 command：

```
/code-review
```

Agent 应该会加载 `workflow-code-review` skill 并按照定义的审查流程执行。

## 推荐工作流

框架会根据上下文自动触发 skill，但**主动通过 command 调用更可靠**。完整的功能开发流程推荐顺序：

```
/requirements-clarification   →  澄清需求，生成 spec.md（AI 作为引导者）
         ↓
/system-design                →  设计方案，填写 spec.md 的设计章节（AI 作为协作者）
         ↓
/code-generation              →  实现代码，自动加载编码规范，逐任务推进（AI 作为执行者）
         ↓
/test-generation              →  基于 spec 和实现生成测试
         ↓
/code-review                  →  对照 spec 和规范审查变更
```

AI 在不同阶段扮演不同角色：需求阶段是**引导者**（通过结构化提问帮你将模糊意图显式化）、设计阶段是**协作者**（分析权衡、提出替代方案）、编码阶段是**执行者**（在明确规格下自主实现）。每个阶段产出的结构化文档自然成为下一阶段的高质量上下文，形成正向循环。

**流程的严格程度与任务的风险成正比**——不必每次都走完整流程：

- **小 bug 修复？** → 直接 `/code-generation`（它会评估复杂度，简单改动可跳过 spec）
- **线上故障？** → 直接 `/troubleshooting`
- **优化热点路径？** → 直接 `/performance-optimization`
- **审查一个 diff？** → 直接 `/code-review`

## 经验沉淀（Self-Refinement）

AI agent 没有跨会话记忆——这一轮对话中纠正过的错误，下一轮可能原样重现。Self-Refinement 机制通过将纠错经验转化为持久化的 Skills 来解决这个问题。

**两种触发方式：**

- **自动触发**：当你在协作中纠正了 AI 的错误，AI 会在完成纠正后，在回复末尾给出轻量的经验沉淀建议（不超过 3 条），经你确认后执行。
- **手动触发（`/reflect`）**：主动发起当前对话的全面回顾。AI 会识别对话中所有被纠正的错误模式，对每个错误执行完整的诊断闭环：识别错误模式 → 诊断根因 → 检索现有知识 → 生成建议 → 用户确认 → 执行更新。

**典型场景：**

| 你纠正了什么 | 沉淀到哪里 |
|------------|-----------|
| "变量命名应该用 snake_case" | `std-*` 编码规范 Skill |
| "这个模块的锁应该用 bthread mutex" | `std-*` 模块规范 Skill |
| "不要跳过 spec 直接写代码" | `workflow-code-generation` Skill |
| "错误处理要用 Status 而不是返回 -1" | `bp-coding-best-practices` Skill |

这使得团队的工程经验可以**从对话中自然生长**，而非依赖人工维护文档。每次纠正都是一次改进框架的机会。

## 如何扩展

Best Practices 和 Standards 是框架中**最需要按项目定制**的部分。框架不规定具体内容——它规定的是**知识应该以什么形式存在、在什么时机被加载、如何被演进**。

### 添加语言规范

创建新的 `std-*` skill：

```
skills/
└── std-python/
    ├── SKILL.md              # 编码规范概览
    └── reference/
        └── style-guide.md    # 详细规则
```

然后在需要它的 workflow skill 中注册。例如在 `workflow-code-generation/SKILL.md` 的「按需加载」表格中添加：

```markdown
| `std-python` Skill | 文件为 `.py` |
```

### 添加最佳实践

创建新的 `bp-*` skill：

```
skills/
└── bp-security/
    ├── SKILL.md
    └── reference/
        └── owasp-top-10.md
```

然后在对应的 workflow skill 中引用。常见的集成点：

| 集成点 | 何时添加 |
|--------|----------|
| `workflow-code-generation` → 步骤 3（加载规范） | 编码时需要遵循的规范 |
| `workflow-code-review/code-reviewer.md` → 审查维度 | Review 时需要检查的维度 |
| `workflow-test-generation` → 测试策略 | 需要考虑的测试类别 |
| `troubleshooting` → 模块专项指南 | 特定领域的排查知识 |

### 添加排查案例

将案例文件放入 troubleshooting skill 的 reference 目录：

```
skills/
└── troubleshooting/
    └── reference/
        └── cases/
            └── <module>/
                └── <case-name>.md
```

### 编写新 Skill

使用 `/skill-authoring` 获取编写指导。核心原则：

- **只写 AI 不知道的内容** — 项目特定信息，而非通用知识
- **SKILL.md 不超过 500 行** — 详细内容放在 `reference/` 文件中
- **引用深度只有一层** — 不要出现 A→B→C 链式引用

完整的编写指南见 `skills/bp-skill-authoring/`。

## 解决什么问题

AI 编码 agent 能力很强，但缺乏纪律性。没有明确的流程约束时，它们往往会：

- 跳过需求澄清，直接写代码——在意图转化链的源头就引入损耗，越往下游代价越大
- 会话中途忘记编码规范——LLM 的工作记忆有限且易失，关键上下文随对话膨胀而被稀释
- 跨会话输出质量不一致——LLM 没有持久记忆，上一次会话中学到的教训下一次全部丢失
- 生成速度远超审查速度——生成成本骤降但验证成本未降，认知过载成为新瓶颈

本框架通过将工程最佳实践编码为 agent 可读的 Skill 文件来系统性地解决这些问题：

| 核心实践 | 解决什么 | 框架中的体现 |
|---------|---------|------------|
| **Context Engineering** | 上下文质量决定输出质量上限 | Spec-First、渐进式披露、按需加载 |
| **AI 全链条参与** | 源头损耗传播最远、修复代价最高 | 从需求澄清到代码审查的完整 Workflow |
| **小任务推进、多层次验证** | 控制 AI 概率性输出的错误累积 | 任务拆解 + 逐步审查 + 多维度验证 |
| **Knowledge as Code** | 团队私有知识是 AI 的知识盲区 | Best Practices / Standards 编码为 Skill |
| **Error-Driven Refinement** | LLM 无跨会话记忆，纠错经验易丢失 | Self-Refinement 自动/手动反馈闭环 |

## 适合谁使用

- **个人开发者**：使用 AI 编码 agent（Claude Code、Codex CLI、CodeBuddy、Cursor、Windsurf 等），希望获得更可靠、结构化的输出。
- **工程团队**：希望标准化 AI agent 在团队代码库中的工作方式——将资深工程师的设计原则和编码规范编码为 Skill，通过 AI 分发给每个成员，拉平团队内部的能力差异。
- **Skill 作者**：希望基于成熟的结构和最佳实践来编写自定义 agent skill。

## 框架架构

```
                                   ┌──────────────┐
                                   │   用户请求    │
                                   └──────┬───────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              Workflow Skills (主入口)                                │
│                                                                                     │
│         需求澄清  ──▶  系统设计  ──▶  代码生成  ──▶  测试生成  ──▶  代码评审               │
│                                                                                     │
└─────────────────────────────────────────┬───────────────────────────────────────────┘
                                          │
                                       按需调用
                                          │
                      ┌───────────────────┼───────────────────┐
                      ▼                   ▼                   ▼
            ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
            │  Best Practices  │ │    Standards     │ │      Docs        │
            │   (通用知识)      │ │  (项目私有知识)     │ │  (Docs as Code)  │
            │                  │ │                  │ │                  │
            │  架构设计         │ │   语言规范         │ │  spec.md         │
            │  编码实践         │ │   模块规范         │ │  设计文档         │
            │  性能优化         │ │   测试规范         │ │  架构决策记录      │
            │  分布式系统        │ │  ...             │ │  ...             │
            │  ...             │ │                  │ │                  │
            └──────────────────┘ └──────────────────┘ └──────────────────┘

┌──────────────────────────────────────┐  ┌──────────────────────────────────────────┐
│  Troubleshooting (独立入口)            │  │  Self-Refinement (反馈闭环)               │
│                                      │  │                                          │
│  编译错误 / 运行异常 /                  │  │  自迭代 Skill (自动触发)                   │
│  流水线报错 / 现网告警                  │  │  + 经验沉淀 Command (手动触发)              │
└──────────────────────────────────────┘  └──────────────────────────────────────────┘
```

框架分为三层文件结构，对应不同的加载时机：

```
agents/      → Subagent 定义（如代码审查中的专项 reviewer）
commands/    → 用户触发的入口（如 /code-generation、/troubleshooting）
skills/      → 详细的工作流和知识定义（按需加载，不消耗常驻上下文）
```

### Agents

Agent 是可独立执行子任务的专项角色，由 workflow skill 按需调度：

| Agent | 职责 |
|-------|------|
| `codebase-researcher` | 深度代码库探索，追踪调用链、分析模块依赖 |
| `performance-reviewer` | 性能专项审查：热路径、内存、锁竞争、算法复杂度 |
| `robustness-reviewer` | 健壮性专项审查：边界条件、错误处理、资源泄漏 |
| `spec-compliance-reviewer` | Spec 符合度审查：验证实现是否匹配设计 |
| `standards-reviewer` | 编码规范审查：命名、风格、宏使用、锁原语 |
| `magical-prompt-reviewer` | 契约与信任链审查：变更引入的契约破坏和信任链断裂 |
| `review-critic` | 对抗性验证：对候选 finding 寻找反证，驳回误报 |

### Commands（用户触发）

Command 是加载对应 skill 的快捷方式：

| Command | 加载的 Skill |
|---------|-------------|
| `/requirements-clarification` | `workflow-requirements-clarification` |
| `/system-design` | `workflow-system-design` |
| `/code-generation` | `workflow-code-generation` |
| `/test-generation` | `workflow-test-generation` |
| `/code-review` | `workflow-code-review` |
| `/troubleshooting` | `troubleshooting` |
| `/performance-optimization` | `bp-performance-optimization` |
| `/reflect` | `self-refinement` |
| `/skill-authoring` | `bp-skill-authoring` |

### Skills

Skill 是框架的核心载体——模块化、按需加载、可组合。大量 Skills 不会拖垮上下文窗口，因为框架采用三层加载机制：

| 层级 | 内容 | 加载时机 | Token 成本 |
|------|------|---------|-----------|
| **L1: Metadata** | YAML frontmatter（name, description） | Agent 启动时 | ~100/Skill |
| **L2: Instructions** | SKILL.md 主体指令 | Skill 被触发时 | < 5k |
| **L3: Resources** | reference/（参考文档）、scripts/（脚本） | 被显式引用时 | 按需 |

Skill 分为四类：

| 前缀 | 类型 | 角色 | 示例 |
|------|------|------|------|
| `workflow-*` | 工作流 | 端到端流程控制，是主入口 | `workflow-code-generation`、`workflow-code-review` |
| `bp-*` | 最佳实践 | 通用工程知识，由工作流按需加载 | `bp-coding-best-practices`、`bp-distributed-systems` |
| `std-*` | 编码规范 | 语言/团队特定的编码标准 | `std-cpp`、`std-go` |
| *（其他）* | 工具型 | 独立能力 | `troubleshooting`、`self-refinement` |

