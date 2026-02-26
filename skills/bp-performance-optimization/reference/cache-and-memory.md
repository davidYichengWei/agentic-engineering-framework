# Cache and Memory Optimization

缓存和内存布局优化——现代 CPU 通常受内存延迟而非指令数限制。

---

## 1. AoS → SoA (Array of Structures → Structure of Arrays)

单字段扫描时显著提升 cache 命中率。

```cpp
// Before: AoS - 访问 mass 时加载无关的 x,y,z
struct Particle { float x, y, z; float mass; };
std::vector<Particle> particles(1000);

float total_mass_aos(const std::vector<Particle>& p) {
  float s = 0;
  for (const auto& pt : p) s += pt.mass;  // Cache 浪费 75%
  return s;
}

// After: SoA - mass 连续存储
struct ParticlesSoA {
  std::vector<float> x, y, z, mass;
};

float total_mass_soa(const ParticlesSoA& p) {
  float s = 0;
  for (float m : p.mass) s += m;  // 完美线性访问
  return s;
}
```

---

## 2. Loop Tiling (Blocking)

将大数据切块，使工作集 fit 进 L1/L2 cache。

```cpp
// Before: 大矩阵遍历导致 cache thrashing
void matmul_naive(const float* A, const float* B, float* C, int N) {
  for (int i = 0; i < N; ++i)
    for (int j = 0; j < N; ++j)
      for (int k = 0; k < N; ++k)
        C[i*N + j] += A[i*N + k] * B[k*N + j];
}

// After: 分块处理，提升 cache 复用
void matmul_tiled(const float* A, const float* B, float* C, int N) {
  constexpr int BS = 32;  // 匹配 L1 cache
  for (int ii = 0; ii < N; ii += BS)
    for (int kk = 0; kk < N; kk += BS)
      for (int jj = 0; jj < N; jj += BS)
        for (int i = ii; i < std::min(ii + BS, N); ++i)
          for (int k = kk; k < std::min(kk + BS, N); ++k) {
            float aik = A[i*N + k];
            for (int j = jj; j < std::min(jj + BS, N); ++j)
              C[i*N + j] += aik * B[k*N + j];
          }
}
```

---

## 3. False Sharing Prevention

多线程时确保独立变量在不同 cache line（通常 64 字节）。

```cpp
// Before: 两个原子变量在同一 cache line，导致 cache 乒乓
struct BadCounters {
  std::atomic<int64_t> thread_a_count;
  std::atomic<int64_t> thread_b_count;
};

// After: 对齐到 cache line
struct GoodCounters {
  alignas(64) std::atomic<int64_t> thread_a_count;
  alignas(64) std::atomic<int64_t> thread_b_count;
};
```

---

## 4. Prefetch

流式访问时提前加载下一批数据。

```cpp
#if defined(__GNUC__) || defined(__clang__)
  #define PREFETCH(p) __builtin_prefetch((p), 0, 3)
#else
  #define PREFETCH(p) ((void)0)
#endif

uint64_t sum_prefetch(const uint64_t* a, size_t n) {
  uint64_t s = 0;
  for (size_t i = 0; i < n; ++i) {
    if (i + 8 < n) PREFETCH(a + i + 8);
    s += a[i];
  }
  return s;
}
```

---

## 5. 用索引替代指针

减少间接访问，提升局部性。

```cpp
// Before: 链表指针追逐
struct NodePtr { int value; NodePtr* next; };

// After: 索引存储，数据连续
struct NodeIdx { int value; int next; };  // next = -1 表示 null

int sum_list(const std::vector<NodeIdx>& nodes, int head) {
  int s = 0;
  for (int i = head; i != -1; i = nodes[i].next)
    s += nodes[i].value;
  return s;
}
```

---

## 6. 内存对齐和行优先访问

```cpp
// Before: 列优先访问（跳跃式）
void bad_order(float* a, int R, int C) {
  for (int j = 0; j < C; ++j)
    for (int i = 0; i < R; ++i)
      a[i*C + j] += 1.0f;  // 每次跳 C 个 float
}

// After: 行优先访问（顺序式）
void good_order(float* a, int R, int C) {
  for (int i = 0; i < R; ++i)
    for (int j = 0; j < C; ++j)
      a[i*C + j] += 1.0f;  // 连续访问
}
```

---

## 7. 减少分配：Reserve 和 Buffer 复用

```cpp
// 预分配容量
std::vector<int> gen(int n) {
  std::vector<int> v;
  v.reserve(n);  // 避免多次 realloc
  for (int i = 0; i < n; ++i) v.push_back(i);
  return v;
}

// 复用临时 buffer
void process_many(const std::vector<std::vector<int>>& inputs) {
  std::vector<int> tmp;  // 一次分配
  for (const auto& in : inputs) {
    tmp.clear();          // 保留容量
    tmp.insert(tmp.end(), in.begin(), in.end());
    // process tmp...
  }
}
```

---

## 8. Arena/Pool Allocator

批量分配，避免频繁 malloc。

```cpp
#include <memory_resource>

void arena_example() {
  std::byte buf[1 << 16];
  std::pmr::monotonic_buffer_resource arena(buf, sizeof(buf));
  
  std::pmr::vector<std::pmr::string> v(&arena);
  v.emplace_back("a");
  v.emplace_back("b");  // 快速 bump 分配，无 malloc
}
```

### Object Pool

```cpp
struct Node { int x; Node* next; };

struct Pool {
  std::vector<Node> storage;
  Node* free_list = nullptr;
  
  explicit Pool(size_t n) : storage(n) {
    for (auto& node : storage) {
      node.next = free_list;
      free_list = &node;
    }
  }
  
  Node* alloc() {
    Node* p = free_list;
    free_list = free_list->next;
    return p;
  }
  
  void dealloc(Node* p) {
    p->next = free_list;
    free_list = p;
  }
};
```
