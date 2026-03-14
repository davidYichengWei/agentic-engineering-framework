---
name: workflow-code-generation
description: 代码文件修改的统一入口。当用户请求任何代码变更（新功能、优化、Bug 修复、重构）时必须首先调用此 skill。仅适用于代码文件（如 .cc/.cpp/.h/.go/.py 等），修改 .md 等非代码文件时不需要调用。它会评估复杂度、检查 spec.md、生成 tasks.md、并逐个任务执行。
---

# 代码生成

> 所有代码修改的统一入口。**先加载规范，再写代码。**

## 工作流程

### 步骤 1：评估复杂度

| 复杂度 | spec 要求 |
|--------|----------|
| **简单**（单文件、局部） | 跳过 spec/tasks.md，走 Fast-Path |
| **中等+**（多文件、跨模块） | 需要 spec + tasks.md |

**中等+** → 查找 `docs/design-docs/<module>/<feature>/spec.md`：
- 存在且完整 → 读取
- 不存在/不完整 → **调用 `workflow-requirements-clarification`**（禁止自行澄清）

**强制**：`spec.md` 与 `tasks.md` 同时存在时，编码前必须完整读取两者。

### Fast-Path：简单修改

1. 确认修改目标与验证方式
2. 加载编码规范（同步骤 3）
3. 实现 → 并行调用 4 个专项 reviewer subagent → 合并评估意见 → 修复循环（同步骤 4 Phase 2）
4. 输出改动说明 + review 评估表格

### 步骤 2：检查/创建 tasks.md

- **已存在** → 找第一个未完成任务，进入步骤 3
- **不存在** → 调用 `task-planner` subagent 生成：

  使用 `Task` 工具调用 `task-planner`，传递：
  - `spec.md` 的完整路径
  - 任务模板参考：[reference/tasks_template.md](reference/tasks_template.md)

  `task-planner` 返回任务清单内容后：
  1. 将内容写入 spec.md 同目录的 `tasks.md`
  2. **展示任务列表给用户确认**
  3. 用户确认后进入步骤 3

### 步骤 3：加载编码规范（强制前置）

> **未加载规范就写代码 → 立即停止，先加载。**

#### 必须加载

| 规范 | 说明 |
|------|------|
| `bp-coding-best-practices` | 通用编码最佳实践 |
| `bp-performance-optimization` | 性能优化（所有代码都是性能敏感的） |

#### 按需加载

| 规范 | 何时加载 |
|------|----------|
| `std-company-cpp` | `.cc`/`.cpp`/`.h` 文件 |
| `std-company-go` | `.go` 文件 |
| `bp-distributed-systems` | 涉及网络通信、多节点协调、一致性、故障恢复 |

### 步骤 4：逐个任务实现

**核心规则：一个 Task → review 通过 → 报告 → 等用户批准 → 下一个 Task**

#### Phase 1: 实现

修改代码，更新 tasks.md 标记 In Progress。

#### Phase 2: 并行多维度 Code Review 与修复循环（强制）

**4a. 并行调用 4 个专项 reviewer subagent**

使用 `Task` 工具**同时**发起 4 个并行调用：

| Reviewer | Subagent | 传递内容 |
|----------|----------|----------|
| **Spec 符合度** | `spec-compliance-reviewer` | `spec.md` 路径、`tasks.md` 路径、当前 Task ID、需审查的文件列表 |
| **编码规范** | `standards-reviewer` | 需加载的 skill 列表（步骤 3 中已加载的所有编码规范 skill）、需审查的文件列表 |
| **性能** | `performance-reviewer` | `spec.md` 路径（可选）、需审查的文件列表 |
| **健壮性** | `robustness-reviewer` | `spec.md` 路径（可选）、需审查的文件列表 |

**4b. 合并 & 去重 4 个 reviewer 的结果**

收到 4 份审查结果后：
1. **去重**：不同 reviewer 指向同一代码位置的相似问题，合并为一条（保留更严重的定级）
2. **统一严重程度**：Error > Warning > Info
3. 汇总为统一的审查结论：任一 reviewer 报告 NEEDS_CHANGES → 整体 NEEDS_CHANGES

**4c. 逐条评估 review 意见**（不可跳过）

对**每条**合并后的 comment，基于 spec/代码证据判断合理性：

**合理** → 反思犯错原因：

| 原因分类 | 含义 |
|---------|------|
| **Spec 理解偏差** | spec 写清楚了但理解错误 |
| **规范未遵守** | 编码规范有要求但未执行 |
| **执行遗漏** | 漏掉边界/细节 |
| **设计考虑不足** | 需更深层设计思考 |

**不合理** → 给出反驳依据（引用代码行或 spec 章节）

输出评估表格：

```markdown
### Review 意见评估

| # | 维度 | 位置 | Review 意见 | 合理？ | 判断依据 | 犯错原因 / 反驳理由 |
|---|------|------|------------|--------|---------|-------------------|
| 1 | Spec/规范/性能/健壮性 | `file:line` | [摘要] | ✅/❌ | [证据] | [分类 + 说明] |

**需修复**: N 条 | **无需修复**: M 条
```

**4d. 自动修复 → 定向 Re-review 循环**

修复合理的 comment → **仅重跑报了问题的维度对应的 reviewer**（定向 re-review）→ 重复 4b-4d，直到所有维度 PASS。

> 例外：如果修复量超过 20 行，升级为全量重跑（4 个 reviewer 全部重跑）。

#### Phase 3: 汇报 → 停止等待

输出报告，更新 tasks.md 标记 Completed，**停止等待用户批准**。

#### Task 完成报告模板

```markdown
---
## ✅ Task N 完成

### 改动文件
- `path/to/file.cc`: [改动说明]

### 改动内容
[做了什么，为什么]

### Code Review 结果

**Review 轮次**: [第 K 轮通过]

#### Review 意见评估（最后一轮）

| # | 维度 | 位置 | Review 意见 | 合理？ | 判断依据 | 犯错原因 / 反驳理由 |
|---|------|------|------------|--------|---------|-------------------|
| 1 | Spec/规范/性能/健壮性 | `file:line` | [摘要] | ✅/❌ | [证据] | [分类 + 说明] |

**需修复**: N 条 | **无需修复**: M 条

---

使用 `ask_followup_question` 工具询问用户，提供以下选项：
- **继续/LGTM** → 下一个任务
- **修改** → 重新提交（用户附带修改要求）
- **回滚** → 撤销本次改动
```

---

## 用户跳过 spec 时

必须生成简化版 spec.md（标注 `Status: Quick Draft`）。**禁止无 spec 修改中等+代码。**

## 恢复中断的任务

读取 `tasks.md` → 找未完成任务 → **重新加载编码规范** → 继续执行。
