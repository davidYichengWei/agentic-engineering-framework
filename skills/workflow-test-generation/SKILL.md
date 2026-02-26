---
name: workflow-test-generation
description: 基于 spec.md 或被测代码生成单元测试、集成测试、性能测试。当用户请求生成测试、采用 TDD 模式、或代码生成完成后需要补充测试时使用。
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
2. 测试计划明确 → 直接生成测试
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
| SQL 功能、存储引擎、复制集群 | 集成测试（MTR） |
| spec.md 有性能指标要求 | 性能测试 |

### Step 2: 分析被测代码

- 识别公共接口（只测公共方法）
- 识别输入、输出、副作用
- 识别依赖（需要 mock 的部分）

### Step 3: 应用边界条件

**必须参考** [reference/boundary-checklist.md](reference/boundary-checklist.md)，选择适用的边界条件。

### Step 4: 制定测试计划

列出要测试的函数/类及对应场景，使用 `todo_write` 创建测试任务清单：

```
示例：
1. [pending] TestClass::MethodA - 正常路径 + 边界条件
2. [pending] TestClass::MethodB - 异常场景
3. [pending] IntegrationTest - 端到端流程
```

**与用户确认**测试计划后再开始生成。

### Step 5: 逐个生成测试

按 todo 逐个生成，每个测试包含三类场景：
1. **正常路径** - happy path
2. **边界条件** - 基于 checklist
3. **异常场景** - 错误输入、异常处理

每完成一个，标记 `[completed]`，运行验证后继续下一个。

---

## 单元测试

### FIRST 原则

| 原则 | 检查点 |
|------|--------|
| **Fast** | 无真实 I/O、网络、数据库 |
| **Independent** | 无共享可变状态 |
| **Repeatable** | 无真实时间、随机数 |
| **Self-Validating** | 明确 Pass/Fail |
| **Timely** | TDD 或与代码同步 |

### AAA 结构

```cpp
// Arrange: 准备输入和 mock
// Act: 执行被测方法（单次调用）
// Assert: 验证输出/状态
```

---

## 性能测试

| 原则 | 说明 |
|------|------|
| **基线对比** | 与历史数据对比 |
| **可复现** | 固定数据集、配置 |
| **预热** | 排除冷启动影响 |
| **多次运行** | 取 P99，避免抖动 |

---

## Mock 原则

| 原则 | 说明 |
|------|------|
| **只 mock 架构边界** | 网络、数据库、文件系统 |
| **不 mock 内部协作者** | 避免测试实现细节 |
| **优先使用 Fake** | 内存实现 > 复杂 mock |

**检验标准**：重构内部实现时，测试不应失败。

---

## 反模式

| 反模式 | 正确做法 |
|--------|----------|
| 测试实现细节 | 测试行为/契约 |
| 测试中有 if/for | 测试应线性简单 |
| Sleep 等待 | 用 fake clock/latch |
| 共享可变状态 | 每个测试独立 setup |
| 弱断言（assertNotNull） | 断言具体值 |

---

## 模块专项参考

| 测试类型 | 参考资料 |
|----------|----------|
| **边界条件** | [reference/boundary-checklist.md](reference/boundary-checklist.md) |

---

## 强制规则

1. **必须覆盖三类场景**：正常路径 + 边界条件 + 异常场景
2. **必须应用边界条件 checklist**
3. **必须可编译运行**
4. **单元测试遵循 FIRST + AAA**
5. **禁止测试私有方法**
6. **禁止 Sleep**
