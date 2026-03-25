---
name: workflow-test-generation
description: 测试生成。基于 spec.md 或被测代码，生成单元测试、集成测试、性能测试。当用户请求生成测试、TDD 模式、或 workflow-code-generation 完成后触发。
---

# 测试生成

## 触发条件

- 用户请求生成测试
- TDD 模式：设计完成后，编码前
- `workflow-code-generation` 完成后，用户选择编写测试

## 工作流程

### Step 0: 检查 spec.md

尝试读取 `docs/design-docs/<module>/<feature>/spec.md`：

**有 spec.md**：
1. 读取 "7. 测试计划"
2. 测试计划明确 → 进入 Step 1
3. 测试计划不完整 → 补充读取 "2. 目标"、"3. 需求"、"4. 设计方案"，自行判断

**无 spec.md**（为已有代码补测试）：

```json
{
  "questions": [{
    "id": "no_spec_action",
    "question": "未找到 spec.md，请选择：",
    "options": [
      "先澄清需求生成 spec.md（推荐）",
      "直接告诉我要测试什么（快速）"
    ],
    "multiSelect": false
  }]
}
```

- 选择 "先澄清需求" → 调用 `workflow-requirements-clarification` skill
- 选择 "直接测试" → 询问要测哪些函数/类，基于代码生成

### Step 1: 确定测试类型

根据代码特征自动判断，**无法判断时才询问**：

| 特征 | 测试类型 |
|------|----------|
| 纯函数、无外部依赖 | 单元测试 |
| 端到端流程、多组件交互 | 集成测试 |
| spec.md 有性能指标要求 | 性能测试 |

### Step 2: 制定测试计划

调用 `test-writer` subagent 分析被测代码，提供以下信息：

```
prompt 模板：
"分析以下被测代码，制定测试计划（不要生成测试代码）：
- spec.md 路径：<path>（如有）
- 被测代码：<文件路径列表>
- 测试类型：<unit/integration/perf>
- spec 测试计划摘要：<摘要>（如有）

请返回：每个需要测试的函数/类、对应的测试场景（正常路径 + 边界条件 + 异常场景）。"
```

**与用户确认**测试计划后再开始生成。

### Step 3: 逐个生成测试

用户确认计划后，对每个测试任务调用 `test-writer` subagent 生成测试代码：

```
prompt 模板：
"为以下目标生成测试代码：
- spec.md 路径：<path>（如有）
- 被测代码：<文件路径列表>
- 测试类型：<unit/integration/perf>
- 当前任务：<具体测试任务描述>
- 测试场景：<正常路径/边界条件/异常场景的具体要求>

请生成完整的、可编译的测试代码。"
```

每完成一个任务，标记 `[completed]`，继续下一个。

**注意**：可以将多个相关的测试任务合并到一次 `test-writer` 调用中（例如同一个类的多个方法），减少调用次数。

---

## 强制规则

1. **Step 0/1/2 的用户交互必须在主 agent 中完成**：不要把需要用户确认的步骤下发给 subagent
2. **测试计划必须经用户确认**：确认后才能开始生成
3. **实际测试生成必须通过 `test-writer` subagent**：不要在主 agent context 中直接写测试代码
4. **prompt 必须提供充分上下文**：spec 路径、被测代码路径、测试类型、具体任务描述

## 反模式

| ❌ 错误做法 | ✅ 正确做法 |
|------------|-----------|
| 在主 agent 中直接写测试代码 | 通过 `test-writer` subagent 生成 |
| 跳过用户确认直接生成 | Step 2 计划确认后才生成 |
| 给 subagent 的 prompt 只说"写个测试" | 提供 spec 路径、文件路径、测试类型、具体场景 |
| 一次把所有任务塞给 subagent | 按任务分批调用，或合并相关任务 |
