# Loop Rules

循环优化技术——热点代码通常在循环中。

---

## 1. Code Motion Out of Loops

将循环不变量移出循环。

### 示例：基本代码外提

```cpp
// Before: strlen() called every iteration
void process_string(const char* str) {
  for (size_t i = 0; i < strlen(str); ++i) {  // O(n) * n = O(n²)
    process_char(str[i]);
  }
}

// After: Compute once outside loop
void process_string(const char* str) {
  const size_t len = strlen(str);  // O(n) once
  for (size_t i = 0; i < len; ++i) {  // O(n) total
    process_char(str[i]);
  }
}
```

### 示例：对象创建外提

```cpp
// Before: Regex compiled every iteration
void match_all(const std::vector<std::string>& lines) {
  for (const auto& line : lines) {
    std::regex pattern("\\d{4}-\\d{2}-\\d{2}");  // Expensive!
    if (std::regex_search(line, pattern)) {
      // ...
    }
  }
}

// After: Compile once
void match_all(const std::vector<std::string>& lines) {
  static const std::regex pattern("\\d{4}-\\d{2}-\\d{2}");
  for (const auto& line : lines) {
    if (std::regex_search(line, pattern)) {
      // ...
    }
  }
}
```

### 示例：计算外提

```cpp
// Before: Division in every iteration
void scale_array(double* arr, size_t n, double divisor) {
  for (size_t i = 0; i < n; ++i) {
    arr[i] = arr[i] / divisor;  // Division is expensive
  }
}

// After: Multiply by reciprocal
void scale_array(double* arr, size_t n, double divisor) {
  const double multiplier = 1.0 / divisor;  // One division
  for (size_t i = 0; i < n; ++i) {
    arr[i] = arr[i] * multiplier;  // Multiplication is cheaper
  }
}
```

---

## 2. Combining Tests (Sentinels)

使用哨兵减少循环内的测试次数。

### 示例：线性搜索哨兵

```cpp
// Before: Two tests per iteration
int find(int* arr, size_t n, int target) {
  for (size_t i = 0; i < n; ++i) {      // Test 1: bounds check
    if (arr[i] == target) return i;      // Test 2: value check
  }
  return -1;
}

// After: One test per iteration using sentinel
int find_sentinel(int* arr, size_t n, int target) {
  if (n == 0) return -1;
  
  int last = arr[n - 1];      // Save last element
  arr[n - 1] = target;        // Place sentinel
  
  size_t i = 0;
  while (arr[i] != target) {  // Only one test!
    ++i;
  }
  
  arr[n - 1] = last;          // Restore last element
  
  if (i < n - 1 || last == target) {
    return i;
  }
  return -1;
}
```

### 示例：字符串搜索哨兵

```cpp
// Before: Check for null terminator and match
const char* find_char(const char* str, char c) {
  while (*str != '\0' && *str != c) {
    ++str;
  }
  return (*str == c) ? str : nullptr;
}

// After: If string is in modifiable buffer with extra space
const char* find_char_sentinel(char* str, size_t len, char c) {
  str[len] = c;  // Place sentinel at end
  while (*str != c) {
    ++str;
  }
  return (str < str + len) ? str : nullptr;
}
```

---

## 3. Loop Unrolling

展开循环减少迭代开销。

### 示例：基本展开

```cpp
// Before: 1000 iterations, each with increment + compare + branch
void sum_array(const int* arr, size_t n) {
  int sum = 0;
  for (size_t i = 0; i < n; ++i) {
    sum += arr[i];
  }
  return sum;
}

// After: 250 iterations (4x unroll)
int sum_array_unrolled(const int* arr, size_t n) {
  int sum = 0;
  size_t i = 0;
  
  // Process 4 elements per iteration
  for (; i + 3 < n; i += 4) {
    sum += arr[i];
    sum += arr[i + 1];
    sum += arr[i + 2];
    sum += arr[i + 3];
  }
  
  // Handle remainder
  for (; i < n; ++i) {
    sum += arr[i];
  }
  
  return sum;
}
```

### 示例：Duff's Device (极端展开)

```cpp
// Copy n bytes from src to dst
void duff_copy(char* dst, const char* src, size_t n) {
  size_t iterations = (n + 7) / 8;
  switch (n % 8) {
    case 0: do { *dst++ = *src++;
    case 7:      *dst++ = *src++;
    case 6:      *dst++ = *src++;
    case 5:      *dst++ = *src++;
    case 4:      *dst++ = *src++;
    case 3:      *dst++ = *src++;
    case 2:      *dst++ = *src++;
    case 1:      *dst++ = *src++;
            } while (--iterations > 0);
  }
}
```

**注意**：现代编译器通常自动展开，手动展开需 Profile 验证收益。

---

## 4. Loop Fusion

合并相同范围的循环。

### 示例：数组处理合并

```cpp
// Before: Two passes over the same data
void process_data(float* data, size_t n) {
  // Pass 1: Normalize
  float max_val = *std::max_element(data, data + n);
  for (size_t i = 0; i < n; ++i) {
    data[i] /= max_val;
  }
  
  // Pass 2: Apply threshold
  for (size_t i = 0; i < n; ++i) {
    if (data[i] < 0.1f) data[i] = 0.0f;
  }
}

// After: Single pass (assuming max is known or computed separately)
void process_data_fused(float* data, size_t n, float max_val) {
  for (size_t i = 0; i < n; ++i) {
    data[i] /= max_val;
    if (data[i] < 0.1f) data[i] = 0.0f;
  }
}
```

### 示例：初始化合并

```cpp
// Before: Separate initialization loops
void init_arrays(int* a, int* b, int* c, size_t n) {
  for (size_t i = 0; i < n; ++i) a[i] = 0;
  for (size_t i = 0; i < n; ++i) b[i] = 1;
  for (size_t i = 0; i < n; ++i) c[i] = i;
}

// After: Fused initialization
void init_arrays_fused(int* a, int* b, int* c, size_t n) {
  for (size_t i = 0; i < n; ++i) {
    a[i] = 0;
    b[i] = 1;
    c[i] = i;
  }
}
```

**优点**：更好的缓存局部性，减少循环开销。

---

## 5. Unconditional Branch Removal

旋转循环将条件分支移到循环底部。

### 示例：While 循环转 Do-While

```cpp
// Before: Jump to test at start
void process(Node* head) {
  Node* p = head;
  while (p != nullptr) {  // Test at top, unconditional jump at bottom
    process_node(p);
    p = p->next;
  }
}

// After: Test at bottom (one fewer jump per iteration)
void process(Node* head) {
  if (head == nullptr) return;
  
  Node* p = head;
  do {
    process_node(p);
    p = p->next;
  } while (p != nullptr);  // Conditional jump only
}
```

### 示例：For 循环底部测试

```cpp
// Before: Standard for loop
for (int i = 0; i < n; ++i) {
  // body
}

// Compiler typically transforms to:
int i = 0;
if (i < n) {
  do {
    // body
    ++i;
  } while (i < n);
}
```

---

## 权衡考虑

| 技术 | 优点 | 代价 |
|------|------|------|
| Code Motion | 减少重复计算 | 可能增加寄存器压力 |
| Sentinels | 减少分支 | 需要可修改内存、边界处理 |
| Unrolling | 减少循环开销 | 代码膨胀、可能伤害 I-cache |
| Fusion | 更好缓存局部性 | 可能增加寄存器压力 |
| Branch Removal | 减少跳转 | 代码可读性下降 |

**最佳实践**：
1. 先让编译器优化（`-O2`, `-O3`）
2. Profile 确认循环是热点
3. 检查编译器是否已应用该优化
4. 手动优化后再次 Profile 验证
