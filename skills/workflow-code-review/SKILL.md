---
name: workflow-code-review
description: 代码评审。协调 5 个专项 reviewer subagent 对代码进行并行多维度审查。可由用户直接触发，也可由主 agent 加载后作为 Judge 执行。
---

# Multi-Agent Code Review

你是 **Judge**——编排流程、去重分诊、最终裁决、输出报告。你不是 reviewer，不产出 finding。

## Subagent 清单

| 角色 | subagent_name | 调用方式 |
|------|---------------|----------|
| 性能审查 | `performance-reviewer` | 始终调用 |
| 健壮性审查 | `robustness-reviewer` | 始终调用 |
| 工程规范审查 | `standards-reviewer` | 始终调用 |
| 契约与信任链审查 | `magical-prompt-reviewer` | 始终调用 |
| 需求/设计符合度审查 | `spec-compliance-reviewer` | 始终调用 |
| 对抗性验证 | `review-critic` | 有 finding 时调用 |

## 工作流

### 1. 解析 review 范围

根据用户输入确定审查文件和 diff 来源。若范围不清，先澄清再继续。

- 指定文件/spec/task → 直接使用
- 给出 git diff/commit → 解析变更文件
- 无具体范围 → `git diff --cached` 或 `git diff HEAD`

### 2. 构建共享上下文

- 在 `docs/design-docs/` 下搜索相关 `spec.md` 和 `tasks.md`
- 确定审查文件、上下文文件（caller/callee/接口定义）
- 根据文件类型和目录确定适用 skill
- 提炼与本次 review 相关的 spec/task 摘要

### 3. 并行分派 reviewer

**并行**调用 reviewer subagent。**必须等待所有 reviewer subagent 返回后才能进入 Step 4**——禁止主 agent 自己产出 finding。

**跳过列表**：调用方可在请求中通过 `skip_reviewers: [name1, name2]` 指定要跳过的 reviewer。未指定时全部调用。

每个 reviewer 的 prompt 按以下模板构建：

```
审查以下代码变更，在你的维度内产出候选 finding。

[Review Scope]
- 审查文件：{files_under_review}
- 上下文文件：{context_files 或 None}
- Spec：{spec_path 或 N/A}
- Tasks：{tasks_path 或 N/A}
- 当前 Task：{task_id 或 N/A}
- 适用 skill：{skill_list}
- 变更摘要：{scope_summary}

[Severity]
- P0：应阻止合入（功能错误、数据错误、崩溃、严重并发错误、与 spec 关键偏离）
- P1：应该修复但不一定阻塞（特定条件触发、影响可控但风险明确）
- P2：改进建议（不影响正确性/稳定性/性能基线）

只报你的维度内的问题。其他维度的线索可以用一行 handoff note 提示。
```

### 4. 去重归类 & 输出 Reviewer 意见汇总

收齐结果后：
- 合并同根因 / 同位置 / 同调用链的 finding，保留最高 severity
- 归类整理所有 finding，为每条分配全局唯一编号 `F-{seq}`

**零 finding 快速路径**：若所有 reviewer 均无正式 finding，直接跳到第 7 步输出 PASS 报告。

完成去重后，**立即向用户输出 Reviewer 意见汇总**（让用户看到各维度的原始审查视角）：

```markdown
---

## 📋 Reviewer 意见汇总

### 性能审查 (performance-reviewer)

- **F-1** [P1]
  - **位置**: `file:line`
  - **问题**: [一句话问题摘要]
  - **证据**: [支撑该问题的关键代码片段/数据/逻辑推理]
- **F-2** [P2]
  - **位置**: `file:line`
  - **问题**: [一句话问题摘要]
  - **证据**: [支撑该问题的关键代码片段/数据/逻辑推理]
- 💡 **Handoff notes**: [该 reviewer 发现但属于其他维度的线索，无则省略此行]

### 健壮性审查 (robustness-reviewer)

- **F-3** [P0]
  - **位置**: `file:line`
  - **问题**: [一句话问题摘要]
  - **证据**: [支撑该问题的关键代码片段/数据/逻辑推理]
- 💡 **Handoff notes**: ...

### 工程规范审查 (standards-reviewer)

（同上格式，无 finding 则显示"✅ 无发现"）

### 契约与信任链审查 (magical-prompt-reviewer)

（同上格式）

### 需求/设计符合度审查 (spec-compliance-reviewer)

（同上格式）

---
```

### 5. 调用 critic & 输出 Critic 意见

将所有 finding 送 critic 做对抗性验证。调用 `review-critic` subagent，**必须等待 critic subagent 返回后才能进入 Step 6**——禁止主 agent 自己做对抗性验证：

```
[Issue 列表]
（逐条列出 F-{seq}、claim、evidence、location、severity、assumptions）

[Review 上下文]
- 相关文件：{files}
- Spec：{spec_path 或 N/A}
- Tasks：{tasks_path 或 N/A}
- 当前 Task：{task_id 或 N/A}
```

收到 critic 结果后，**立即向用户输出 Critic 意见**：

```markdown
---

## 🔍 Critic 对抗性验证

- **F-1** ✅ 成立
  - **问题**: [reviewer 发现的问题简述]
  - **理由**: [为什么同意 reviewer，补充验证证据]
- **F-2** ❌ 驳回
  - **问题**: [reviewer 发现的问题简述]
  - **理由**: [反证摘要：为什么不成立]
- **F-3** ⚠️ 降级
  - **问题**: [reviewer 发现的问题简述]
  - **理由**: [部分成立但严重度应降低的理由]

---
```

> Critic 结论类型：✅ 成立（同意 reviewer）、❌ 驳回（提供反证）、⚠️ 降级（部分成立但建议降低 severity）。

### 6. 最终裁决

**主 agent 必须亲自调研后裁决**——不能简单采信 reviewer 或 critic 的结论。对每条 issue：

1. **独立调研**：阅读相关代码上下文（调用方、被调用方、数据流）、spec 设计意图、相关注释和 git history，形成自己对该问题的理解
2. **交叉验证**：将 reviewer 提出的证据、critic 的反证与自己调研的结果三方对比
3. **基于证据裁决**：
   - **keep**：问题成立，按 P0/P1/P2 分类进入报告正式问题区
   - **drop**：经调研确认 critic 反证成立或证据不足，丢弃
   - **follow-up note**：不够正式 finding 但值得提醒，进入报告 Follow-up Notes 区（不分级）

> ⚠️ 裁决理由必须引用具体的代码位置、spec 条目或上下文事实，禁止使用"证据充分""证据不足"等空泛表述。

通过门槛：
- 存在 keep 的 P0/P1 → `NEEDS_CHANGES`
- 无 keep 的 P0/P1 → `PASS`
- P2 不阻塞通过

### 7. 输出最终报告

按以下模板输出（整个系统唯一固定格式）：

```markdown
# Code Review 报告

## 审查范围
- **Spec**: [路径 或 N/A]
- **Tasks**: [路径 或 N/A]
- **当前 Task**: [ID/名称 或 N/A]
- **审查文件**: [文件列表]

## 总体结论: PASS / NEEDS_CHANGES

## 裁决明细

> 对每条候选 finding 的最终处置和理由，完整展示审查过程的透明度。

- **F-1** [reviewer名 · 原始优先级] → ✅/❌/⚠️ Critic 结论 → **最终处置 (keep/drop/降级/follow-up)**
  - 裁决依据：[简述经调研后认定成立或不成立的理由]
- **F-2** [reviewer名 · 原始优先级] → ✅/❌/⚠️ Critic 结论 → **最终处置**
  - 裁决依据：[简述理由]
- ...

## 正式问题

### P0（必须修复）

#### P0-1: [问题标题]
- **维度**: [来源维度]
- **位置**: `file:line`
- **问题**: [描述]
- **证据**: [关键证据]
- **建议**: [修复方式]

### P1（应该修复）
...

### P2（建议改进）
...

## Follow-up Notes
- [少量不够进入正式 finding 但值得提醒的事项]
```
