---
name: bp-performance-optimization
description: 提供性能优化方法论、设计原则和具体优化规则。在编写代码、优化性能瓶颈、进行 code review 时使用，涵盖 CPU、内存、I/O、并发等维度。
---

# 性能优化技术

## 前置检查

> **本 skill 是参考资料，不是工作流入口。**

在应用本 skill 的优化规则前，必须确认：

1. **优化目标是否明确？**
   - 具体要优化什么？延迟？吞吐？内存？
   - 当前数据是多少？目标是多少？
   - 如何测量和验证？

2. **是否已经定位瓶颈？**
   - 有 profiling 数据吗？
   - 热点在哪里？

**如果以上问题不清楚**：
- 涉及代码修改 → 先走 `workflow-code-generation` Skill（它会引导需求澄清）
- 仅做分析 → 先与用户确认优化目标和当前数据

---

## 核心理念

> **优化的本质是简化。** 复杂的代码通常更大、更慢、更耗能。

## 方法论

| 阶段 | 要点 |
|------|------|
| **1. 定义目标** | 明确成功标准：延迟？吞吐？内存？功耗？ |
| **2. 设计评估** | 糟糕的设计无法靠优化弥补 |
| **3. Profile** | 关注消耗指标（指令数、cache miss），不只是时间 |
| **4. 算法优先** | 先优化数据结构和算法 |
| **5. 应用规则** | 对热点应用具体优化规则 |

## 设计原则

| 原则 | 说明 |
|------|------|
| **简单直接** | 避免过度抽象；能删代码比加代码好 |
| **连续优于分散** | `vector`/数组优于链表（cache 友好） |
| **直接优于间接** | 减少指针追逐，每次间接访问都是潜在 cache miss |
| **独占优于共享** | `unique_ptr` 无开销；`shared_ptr` 有原子操作开销 |
| **扁平优于深层** | 避免复杂继承层次和虚函数派发 |
| **Early Binding** | 把工作提前做一次，避免重复做多次 |
| **模块化** | 清晰契约，允许局部替换而不改架构 |

---

## 优化规则速查

### Space-for-Time（空间换时间）

| 规则 | 说明 |
|------|------|
| **Data Structure Augmentation** | 添加冗余信息加速操作 |
| **Precomputation** | 预计算并存储结果 |
| **Caching** | 缓存频繁访问的数据 |
| **Lazy Evaluation** | 延迟计算直到真正需要 |

详见 [reference/space-for-time.md](reference/space-for-time.md)

### Time-for-Space（时间换空间）

| 规则 | 说明 |
|------|------|
| **Packing** | 紧凑存储减少内存占用 |
| **Overlaying** | 复用内存空间 |
| **Interpreters** | 用解释器压缩程序表示 |

详见 [reference/time-for-space.md](reference/time-for-space.md)

### Loop Rules（循环优化）

| 规则 | 说明 |
|------|------|
| **Code Motion** | 将循环不变量移出循环 |
| **Combining Tests** | 合并测试条件，使用哨兵 |
| **Loop Unrolling** | 展开循环减少迭代开销 |
| **Loop Fusion** | 合并相同范围的循环 |

详见 [reference/loop-rules.md](reference/loop-rules.md)

### Logic Rules（逻辑优化）

| 规则 | 说明 |
|------|------|
| **Algebraic Identities** | 用等价的更廉价表达式替换 |
| **Short-Circuiting** | 提前终止求值 |
| **Reordering Tests** | 廉价/常成功的测试放前面 |
| **Precompute Logical Functions** | 查表替代逻辑计算 |

详见 [reference/logic-rules.md](reference/logic-rules.md)

### Procedure Rules（过程优化）

| 规则 | 说明 |
|------|------|
| **Inlining** | 内联展开减少调用开销 |
| **Exploit Common Cases** | 快速路径处理常见情况 |
| **Tail Recursion Removal** | 尾递归转循环 |
| **Parallelism** | 利用硬件并行能力 |

详见 [reference/procedure-rules.md](reference/procedure-rules.md)

### Expression Rules（表达式优化）

| 规则 | 说明 |
|------|------|
| **Compile-Time Initialization** | 编译期初始化 |
| **Strength Reduction** | 用廉价操作替代昂贵操作 |
| **Common Subexpression Elimination** | 消除公共子表达式 |
| **Word Parallelism** | 利用字宽并行（位操作） |

详见 [reference/expression-rules.md](reference/expression-rules.md)

### Cache & Memory（缓存和内存）

| 规则 | 说明 |
|------|------|
| **AoS → SoA** | 单字段扫描时结构体数组转数组结构体 |
| **Loop Tiling** | 分块处理，使工作集 fit 进 cache |
| **False Sharing** | 多线程变量对齐到不同 cache line |
| **Arena/Pool** | 批量分配，避免频繁 malloc |

详见 [reference/cache-and-memory.md](reference/cache-and-memory.md)

### Modern C++ Tricks

| 规则 | 说明 |
|------|------|
| **string_view/span** | 避免拷贝，零开销抽象 |
| **Branchless** | 减少分支预测失败惩罚 |
| **CRTP** | 静态多态替代虚函数 |
| **Sharded Lock** | 分片锁减少竞争 |

详见 [reference/modern-cpp-tricks.md](reference/modern-cpp-tricks.md)

---

## Code Review 检查清单

设计层面：
- [ ] 数据结构是否连续紧凑？
- [ ] 是否有不必要的抽象层或间接访问？
- [ ] 关键路径是否简单直接？

实现层面：
- [ ] 循环不变量是否已移出？
- [ ] 是否有可预计算的值？
- [ ] 测试条件顺序是否最优？
- [ ] 是否存在重复计算？
