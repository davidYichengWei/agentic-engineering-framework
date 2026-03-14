---
name: bp-coding-best-practices
description: 通用编码最佳实践。在编写或 review 代码时使用。涵盖可读性、命名、函数设计、控制流、资源安全、注释规范。
---

# 通用编码最佳实践

**设计原则（SOLID、设计模式）**：参见 `bp-component-design` Skill
**特定语言/模块规范**：参见相应的 standards skills

---

## 命名

| 原则 | 说明 |
|------|------|
| **自解释** | `retryCount` 而非 `n` |
| **无魔法数字** | `const int SECONDS_IN_DAY = 86400;` |
| **布尔命名** | `isValid`, `hasAccess`（问题形式） |
| **作用域匹配** | 小作用域可短（`i`），大作用域要描述性 |

---

## 函数设计

| 原则 | 说明 |
|------|------|
| **单一职责** | 一个函数做一件事；名字需要 "And" 说明做太多了 |
| **参数精简** | 超过 3-4 个参数 → 考虑结构体封装 |
| **const 正确** | 不修改的参数标 `const`，防止意外修改 |

---

## 控制流

**Guard Clause**：失败情况先处理并返回，主逻辑保持左对齐

**Early Return**：显式采用 early return 编程范式，尽量将可 early return 的检查前置。

```cpp
// ❌ 深层嵌套
if (order != nullptr) {
    if (order->isValid()) {
        if (order->hasItems()) {
            // main logic
        }
    }
}

// ✅ Guard Clause
if (order == nullptr) return;
if (!order->isValid()) return;
if (!order->hasItems()) return;
// main logic (not nested)
```

---

## 资源安全

| 原则 | 说明 |
|------|------|
| **RAII** | 资源生命周期绑定对象生命周期，避免手动清理分散在多条路径 |
| **所有权显式** | 区分 owner 与 borrower，避免隐式转移所有权 |
| **窄作用域** | 变量声明靠近首次使用，减少悬空与误用概率 |

跨语言场景统一要求：新增分支/返回路径时，必须检查资源契约是否闭环（释放类资源 + 触发类资源）。

### 新增返回路径的契约检查

当新增 `return`、`early exit` 或新分支时，**必须逐一检查函数入口处获取的所有"契约性资源"**。

**契约性资源**：函数持有但不拥有、需要在特定时机交还/触发的资源：
- Closure/Callback（需要 Run）
- 锁（需要 Unlock）
- 引用计数（需要 Release）
- 事务上下文（需要 Commit/Rollback/清理）
- 幂等标记/Nonce（需要 Complete）
- 注册到外部管理器的对象（需要 Remove/Unregister）

**检查方法**：
1. **识别**：在函数开头找所有"获取但需要交还"的东西
2. **对照**：找一个功能相似的现有返回路径，逐行对比它处理了哪些资源
3. **分类**：这个新路径是成功、失败、还是**新的第三种状态**？现有 ownership 注释是否覆盖？

| ❌ 反例 | ✅ 正例 |
|--------|--------|
| 新分支只清理了数据结构，忘了 callback 的执行契约 | 对照已有的 early return 路径，发现它调用了 `callback->Run()`，新路径也需要 |
| 假设"返回成功后调用方会处理 closure" | 检查调用方逻辑，确认 closure 执行责任的真实归属 |
| 只关注"要释放什么"，忽略"要触发什么" | 同时检查释放类资源（锁、内存）和触发类资源（回调、事件） |

---

## 注释

| 场景 | 做法 |
|------|------|
| **何时写** | 仅当意图不明显时；复杂算法；公共 API |
| **写什么** | **Why**（为什么这样做），不是 What（做了什么） |
| **TODO** | 包含上下文和负责人 |

```cpp
// ❌ 复述代码
// Increment i by 1
++i;

// ✅ 解释意图
// Skip index 0 because it is the sentinel slot.
for (size_t i = 1; i < slots.size(); ++i) { ... }
```

---

## 可观测性（日志）

| 场景 | 做法 |
|------|------|
| **关键分支覆盖** | 至少覆盖无数据快速返回、异常状态转换、错误返回三个分支 |
| **级别选择** | `DEBUG` 记录成功路径和排障上下文，`WARN` 记录异常但可恢复路径，`ERROR` 记录失败路径 |
| **上下文信息** | 日志中携带最小必要上下文（如 request_id、key、error_code），避免无上下文日志 |

---

## 进阶

- **可读性详细规则**：[reference/readability.md](reference/readability.md)
- **安全性详细规则**：[reference/safety.md](reference/safety.md)
