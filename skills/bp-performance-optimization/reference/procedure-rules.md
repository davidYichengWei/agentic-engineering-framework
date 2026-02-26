# Procedure Rules

过程优化技术——优化抽象边界。

---

## 1. Collapsing Procedure Hierarchies (Inlining)

内联展开减少调用开销。

### 示例：简单内联

```cpp
// Before: Function call overhead
inline int square(int x) { return x * x; }

int sum_of_squares(const int* arr, size_t n) {
  int sum = 0;
  for (size_t i = 0; i < n; ++i) {
    sum += square(arr[i]);  // Function call each iteration
  }
  return sum;
}

// After: Manual inline (compiler usually does this)
int sum_of_squares(const int* arr, size_t n) {
  int sum = 0;
  for (size_t i = 0; i < n; ++i) {
    sum += arr[i] * arr[i];  // No call overhead
  }
  return sum;
}
```

### 示例：Lambda 内联

```cpp
// Before: std::function has overhead
std::function<int(int)> get_transform(bool negate) {
  if (negate) return [](int x) { return -x; };
  else return [](int x) { return x; };
}

// After: Template for zero-overhead abstraction
template<typename F>
void apply_transform(int* data, size_t n, F transform) {
  for (size_t i = 0; i < n; ++i) {
    data[i] = transform(data[i]);  // Inlined by compiler
  }
}

// Usage: lambda is inlined
apply_transform(data, n, [](int x) { return -x; });
```

---

## 2. Exploit Common Cases

快速路径处理常见情况。

### 示例：快速路径

```cpp
// Before: Always do full validation
bool validate_email(const std::string& email) {
  // Complex regex validation
  static const std::regex pattern(R"([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})");
  return std::regex_match(email, pattern);
}

// After: Quick checks first, expensive validation only if needed
bool validate_email(const std::string& email) {
  // Fast path: quick rejection for common invalid inputs
  if (email.empty()) return false;
  if (email.length() < 5) return false;  // a@b.c minimum
  if (email.find('@') == std::string::npos) return false;
  if (email.find('.') == std::string::npos) return false;
  
  // Slow path: full validation only for candidates
  static const std::regex pattern(R"([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})");
  return std::regex_match(email, pattern);
}
```

### 示例：Small String Optimization (SSO)

```cpp
// Standard library strings use SSO
// Small strings stored inline, no heap allocation
class SmallString {
  static constexpr size_t kInlineCapacity = 15;
  
  union {
    char inline_buffer_[kInlineCapacity + 1];
    struct {
      char* ptr_;
      size_t size_;
      size_t capacity_;
    } heap_;
  };
  bool is_small_;

 public:
  void assign(const char* s, size_t len) {
    if (len <= kInlineCapacity) {
      // Fast path: inline storage
      std::memcpy(inline_buffer_, s, len);
      inline_buffer_[len] = '\0';
      is_small_ = true;
    } else {
      // Slow path: heap allocation
      heap_.ptr_ = new char[len + 1];
      std::memcpy(heap_.ptr_, s, len + 1);
      heap_.size_ = len;
      is_small_ = false;
    }
  }
};
```

---

## 3. Coroutines

将多遍算法转为单遍。

### 示例：生产者-消费者

```cpp
// Before: Two passes - first produce all, then consume all
std::vector<int> produce_all() {
  std::vector<int> results;
  for (int i = 0; i < 1000000; ++i) {
    results.push_back(expensive_compute(i));
  }
  return results;  // Large memory allocation
}

void consume(const std::vector<int>& data) {
  for (int x : data) process(x);
}

// After: Generator pattern (C++20 coroutines or manual state machine)
class Generator {
  int current_ = 0;
  int max_;
  
 public:
  Generator(int max) : max_(max) {}
  
  bool has_next() const { return current_ < max_; }
  
  int next() {
    return expensive_compute(current_++);
  }
};

void process_stream() {
  Generator gen(1000000);
  while (gen.has_next()) {
    process(gen.next());  // Process one at a time, no bulk storage
  }
}
```

### 示例：C++20 Coroutine

```cpp
#include <coroutine>
#include <generator>  // C++23

std::generator<int> generate_values(int max) {
  for (int i = 0; i < max; ++i) {
    co_yield expensive_compute(i);
  }
}

void process_stream() {
  for (int value : generate_values(1000000)) {
    process(value);
  }
}
```

---

## 4. Tail Recursion Removal

尾递归转循环。

### 示例：阶乘

```cpp
// Before: Recursive (stack grows)
uint64_t factorial(int n) {
  if (n <= 1) return 1;
  return n * factorial(n - 1);  // Not tail recursive
}

// Tail recursive form
uint64_t factorial_tail(int n, uint64_t acc = 1) {
  if (n <= 1) return acc;
  return factorial_tail(n - 1, n * acc);  // Tail call
}

// After: Iterative (constant stack)
uint64_t factorial_iter(int n) {
  uint64_t result = 1;
  while (n > 1) {
    result *= n;
    --n;
  }
  return result;
}
```

### 示例：链表遍历

```cpp
// Before: Recursive (stack overflow on long lists)
int sum_list(Node* head) {
  if (head == nullptr) return 0;
  return head->value + sum_list(head->next);
}

// After: Iterative
int sum_list(Node* head) {
  int sum = 0;
  while (head != nullptr) {
    sum += head->value;
    head = head->next;
  }
  return sum;
}
```

### 示例：树遍历（显式栈）

```cpp
// Before: Recursive inorder traversal
void inorder(TreeNode* root, std::vector<int>& result) {
  if (!root) return;
  inorder(root->left, result);
  result.push_back(root->value);
  inorder(root->right, result);
}

// After: Iterative with explicit stack
void inorder_iter(TreeNode* root, std::vector<int>& result) {
  std::stack<TreeNode*> stack;
  TreeNode* current = root;
  
  while (current || !stack.empty()) {
    while (current) {
      stack.push(current);
      current = current->left;
    }
    current = stack.top();
    stack.pop();
    result.push_back(current->value);
    current = current->right;
  }
}
```

---

## 5. Parallelism

利用硬件并行能力。

### 示例：SIMD 向量化

```cpp
// Before: Scalar processing
void add_arrays(const float* a, const float* b, float* c, size_t n) {
  for (size_t i = 0; i < n; ++i) {
    c[i] = a[i] + b[i];
  }
}

// After: SIMD (compiler auto-vectorization or explicit)
#include <immintrin.h>

void add_arrays_simd(const float* a, const float* b, float* c, size_t n) {
  size_t i = 0;
  
  // Process 8 floats at a time with AVX
  for (; i + 7 < n; i += 8) {
    __m256 va = _mm256_loadu_ps(a + i);
    __m256 vb = _mm256_loadu_ps(b + i);
    __m256 vc = _mm256_add_ps(va, vb);
    _mm256_storeu_ps(c + i, vc);
  }
  
  // Remainder
  for (; i < n; ++i) {
    c[i] = a[i] + b[i];
  }
}
```

### 示例：多线程

```cpp
// Before: Single-threaded
void process_data(std::vector<Item>& items) {
  for (auto& item : items) {
    item.result = expensive_compute(item);
  }
}

// After: Parallel processing
#include <execution>

void process_data_parallel(std::vector<Item>& items) {
  std::for_each(std::execution::par_unseq, 
                items.begin(), items.end(),
                [](Item& item) {
                  item.result = expensive_compute(item);
                });
}
```

---

## 权衡考虑

| 技术 | 优点 | 代价 |
|------|------|------|
| Inlining | 消除调用开销 | 代码膨胀、编译时间 |
| Common Cases | 快速处理热路径 | 代码复杂度 |
| Coroutines | 减少内存、单遍处理 | 代码复杂度、调试难 |
| Tail Recursion | 常量栈空间 | 可读性 |
| Parallelism | 多核利用 | 同步开销、复杂度 |

**最佳实践**：
1. 让编译器做 inline 决策（`-O2` 通常足够）
2. 只对 Profile 确认的热路径手动优化
3. 递归深度大于 1000 考虑转迭代
4. 并行化需要足够的工作量来摊销开销
