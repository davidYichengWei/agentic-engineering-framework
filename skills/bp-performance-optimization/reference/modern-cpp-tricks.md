# Modern C++ Performance Tricks

现代 C++ 性能技巧——利用语言特性减少开销。

---

## 1. 避免拷贝：`string_view` 和 `span`

```cpp
#include <string_view>
#include <span>

// string_view: 避免子串分配
bool has_prefix(std::string_view s, std::string_view prefix) {
  return s.size() >= prefix.size() && 
         s.substr(0, prefix.size()) == prefix;  // substr 返回 view，无分配
}

// span: 避免 vector 拷贝
int dot(std::span<const int> a, std::span<const int> b) {
  int s = 0;
  for (size_t i = 0; i < a.size(); ++i) s += a[i] * b[i];
  return s;
}
```

---

## 2. Move 语义和 Emplace

```cpp
// Move 避免深拷贝
std::vector<std::string> build() {
  std::vector<std::string> v;
  std::string tmp(1000, 'x');
  v.push_back(std::move(tmp));  // 移动 buffer，非拷贝
  return v;                      // NRVO 或 move
}

// Emplace 原地构造
std::unordered_map<int, std::string> m;
m.try_emplace(42, 1000, 'x');  // 直接在 map 内构造 string
```

---

## 3. 快速解析：`from_chars`

避免 iostream 和 locale 开销。

```cpp
#include <charconv>

int parse_int(std::string_view s) {
  int x = 0;
  auto [ptr, ec] = std::from_chars(s.data(), s.data() + s.size(), x);
  return (ec == std::errc{}) ? x : 0;
}
```

---

## 4. Branchless 编程

减少分支预测失败的惩罚。

```cpp
// Before: 分支（随机数据时预测失败率高）
int sum_if(const int* a, int n, int threshold) {
  int s = 0;
  for (int i = 0; i < n; ++i)
    if (a[i] > threshold) s += a[i];
  return s;
}

// After: Branchless
int sum_branchless(const int* a, int n, int threshold) {
  int s = 0;
  for (int i = 0; i < n; ++i)
    s += (a[i] > threshold) * a[i];  // 无分支
  return s;
}

// Branchless abs
int branchless_abs(int x) {
  int m = x >> 31;        // 负数时全 1
  return (x ^ m) - m;
}
```

---

## 5. Branch Hints

```cpp
int parse_fast(const char* p) {
  if (p && *p) [[likely]] {
    return static_cast<unsigned char>(*p);
  }
  return 0;  // cold path
}
```

---

## 6. 避免虚函数开销

### CRTP（静态多态）

```cpp
template<typename Derived>
struct Algo {
  int run(int x) { 
    return static_cast<Derived*>(this)->impl(x);  // 编译期绑定
  }
};

struct Fast : Algo<Fast> { 
  int impl(int x) { return x * 2; } 
};
```

### `final` 关键字

```cpp
struct Base { virtual int f(int) = 0; virtual ~Base() = default; };
struct Derived final : Base {  // final 允许编译器 devirtualize
  int f(int x) override { return x + 1; }
};
```

---

## 7. 避免 `std::function` 开销

```cpp
// Before: std::function 有类型擦除开销
void apply(const std::vector<int>& v, std::function<int(int)> f);

// After: 模板参数，编译器可内联
template<typename F>
int transform_sum(const std::vector<int>& v, F f) {
  int s = 0;
  for (int x : v) s += f(x);  // f 被内联
  return s;
}

transform_sum(v, [](int x) { return x * 2; });
```

---

## 8. 并发优化

### Relaxed Atomics

```cpp
// 计数器不需要 seq_cst
void hot_inc(std::atomic<uint64_t>& x) {
  x.fetch_add(1, std::memory_order_relaxed);
}
```

### Sharded Locking

```cpp
struct ShardedMap {
  static constexpr int kShards = 64;
  std::array<std::mutex, kShards> mu;
  std::array<std::unordered_map<int, int>, kShards> m;
  
  void put(int k, int v) {
    int shard = k % kShards;
    std::lock_guard lk(mu[shard]);
    m[shard][k] = v;
  }
};
```

### Read-Heavy: `shared_mutex`

```cpp
struct ReadMostly {
  std::unordered_map<int, int> m;
  mutable std::shared_mutex mu;
  
  int get(int k) const {
    std::shared_lock lk(mu);  // 读锁，不互斥
    auto it = m.find(k);
    return it != m.end() ? it->second : 0;
  }
  
  void put(int k, int v) {
    std::unique_lock lk(mu);  // 写锁，互斥
    m[k] = v;
  }
};
```

### Batch Work

```cpp
struct Batcher {
  std::mutex mu;
  std::vector<int> q;
  
  void push_many(const int* a, int n) {
    std::lock_guard lk(mu);
    q.insert(q.end(), a, a + n);  // 一次锁，多次插入
  }
};
```

---

## 9. 用 Table-Driven 替代分支

```cpp
using Op = int(*)(int, int);
int add(int a, int b) { return a + b; }
int sub(int a, int b) { return a - b; }

int apply(uint8_t opcode, int a, int b) {
  static constexpr std::array<Op, 256> table = [] {
    std::array<Op, 256> t{};
    t[0x01] = add;
    t[0x02] = sub;
    return t;
  }();
  return table[opcode] ? table[opcode](a, b) : 0;
}
```

---

## 10. IO 优化

```cpp
// 禁用 iostream 同步
void fast_io_setup() {
  std::ios::sync_with_stdio(false);
  std::cin.tie(nullptr);
}

// 用 '\n' 不用 std::endl（避免 flush）
std::cout << "line\n";
```
