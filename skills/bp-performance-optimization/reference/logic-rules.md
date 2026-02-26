# Logic Rules

逻辑优化技术——用执行速度换取代码清晰度。

---

## 1. Exploit Algebraic Identities

用等价的更廉价表达式替换昂贵表达式。

### 示例：平方根比较

```cpp
// Before: Expensive sqrt
bool is_within_distance(Point a, Point b, double max_dist) {
  double dx = a.x - b.x;
  double dy = a.y - b.y;
  return sqrt(dx * dx + dy * dy) <= max_dist;
}

// After: Compare squared values (avoid sqrt)
bool is_within_distance(Point a, Point b, double max_dist) {
  double dx = a.x - b.x;
  double dy = a.y - b.y;
  return dx * dx + dy * dy <= max_dist * max_dist;
}
```

### 示例：绝对值比较

```cpp
// Before: Two comparisons
bool is_in_range(int x, int center, int delta) {
  return x >= center - delta && x <= center + delta;
}

// After: Single unsigned comparison (two's complement trick)
bool is_in_range(int x, int center, int delta) {
  return static_cast<unsigned>(x - center + delta) <= 
         static_cast<unsigned>(2 * delta);
}
```

### 示例：模运算替换

```cpp
// Before: Expensive modulo
int wrap_index(int i, int size) {
  return i % size;
}

// After: Bitwise AND (when size is power of 2)
int wrap_index(int i, int size) {
  // Assumes size is power of 2
  return i & (size - 1);
}
```

---

## 2. Short-Circuiting Monotone Functions

提前终止单调函数求值。

### 示例：早期退出

```cpp
// Before: Always compute full result
int count_matching(const std::vector<Item>& items, 
                   const Predicate& pred,
                   int threshold) {
  int count = 0;
  for (const auto& item : items) {
    if (pred(item)) ++count;
  }
  return count >= threshold;
}

// After: Stop when threshold reached
bool has_enough_matching(const std::vector<Item>& items,
                         const Predicate& pred,
                         int threshold) {
  int count = 0;
  for (const auto& item : items) {
    if (pred(item)) {
      if (++count >= threshold) return true;  // Early exit
    }
  }
  return false;
}
```

### 示例：ALL/ANY 短路

```cpp
// Before: Check all elements
bool all_positive(const std::vector<int>& v) {
  bool result = true;
  for (int x : v) {
    if (x <= 0) result = false;  // Continues checking
  }
  return result;
}

// After: Short-circuit on first failure
bool all_positive(const std::vector<int>& v) {
  for (int x : v) {
    if (x <= 0) return false;  // Immediate exit
  }
  return true;
}
```

---

## 3. Reordering Tests

将廉价/常成功的测试放在前面。

### 示例：条件重排序

```cpp
// Before: Expensive check first
bool is_valid_user(User* user) {
  return validate_signature(user) &&  // Expensive crypto op
         user != nullptr &&            // Cheap
         user->is_active();            // Medium
}

// After: Cheap checks first
bool is_valid_user(User* user) {
  return user != nullptr &&            // Cheap, often fails
         user->is_active() &&          // Medium
         validate_signature(user);     // Expensive, only if needed
}
```

### 示例：频率重排序

```cpp
// Before: Alphabetical order
std::string get_day_name(int day) {
  switch (day) {
    case 1: return "Monday";
    case 2: return "Tuesday";
    case 3: return "Wednesday";
    case 4: return "Thursday";
    case 5: return "Friday";
    case 6: return "Saturday";
    case 7: return "Sunday";
  }
}

// After: Order by frequency (if weekdays are most common)
std::string get_day_name(int day) {
  // Assuming mostly weekday queries
  switch (day) {
    case 1: return "Monday";
    case 2: return "Tuesday";
    case 3: return "Wednesday";
    case 4: return "Thursday";
    case 5: return "Friday";
    // Weekend less common
    case 6: return "Saturday";
    case 7: return "Sunday";
  }
}

// Or use jump table for O(1) access
static const char* day_names[] = {
  nullptr, "Monday", "Tuesday", "Wednesday",
  "Thursday", "Friday", "Saturday", "Sunday"
};
std::string get_day_name(int day) {
  return day_names[day];
}
```

---

## 4. Precompute Logical Functions

用查表替代逻辑计算。

### 示例：字符分类表

```cpp
// Before: Multiple comparisons
bool is_identifier_char(char c) {
  return (c >= 'a' && c <= 'z') ||
         (c >= 'A' && c <= 'Z') ||
         (c >= '0' && c <= '9') ||
         c == '_';
}

// After: Lookup table
static const bool is_id_char[256] = {
  // Initialize once with all identifier chars marked true
  ['a'...'z'] = true,
  ['A'...'Z'] = true,
  ['0'...'9'] = true,
  ['_'] = true,
};

bool is_identifier_char(char c) {
  return is_id_char[static_cast<unsigned char>(c)];
}
```

### 示例：状态转换表

```cpp
// Before: Nested conditionals
State next_state(State current, Event event) {
  if (current == IDLE) {
    if (event == START) return RUNNING;
    if (event == STOP) return IDLE;
  } else if (current == RUNNING) {
    if (event == START) return RUNNING;
    if (event == STOP) return IDLE;
    if (event == PAUSE) return PAUSED;
  }
  // ... more states
}

// After: Transition table
static const State transition_table[NUM_STATES][NUM_EVENTS] = {
  /*           START     STOP      PAUSE */
  /* IDLE */   {RUNNING,  IDLE,     IDLE},
  /* RUNNING */{RUNNING,  IDLE,     PAUSED},
  /* PAUSED */ {RUNNING,  IDLE,     PAUSED},
};

State next_state(State current, Event event) {
  return transition_table[current][event];
}
```

---

## 5. Boolean Variable Elimination

用分支替代布尔变量赋值。

### 示例：消除布尔标志

```cpp
// Before: Boolean flag determines later behavior
void process(const std::vector<int>& data, bool ascending) {
  std::vector<int> sorted = data;
  if (ascending) {
    std::sort(sorted.begin(), sorted.end());
  } else {
    std::sort(sorted.begin(), sorted.end(), std::greater<int>());
  }
  
  for (int x : sorted) {
    // Process...
  }
}

// After: Separate functions, no runtime boolean check
void process_ascending(const std::vector<int>& data) {
  std::vector<int> sorted = data;
  std::sort(sorted.begin(), sorted.end());
  for (int x : sorted) { /* Process */ }
}

void process_descending(const std::vector<int>& data) {
  std::vector<int> sorted = data;
  std::sort(sorted.begin(), sorted.end(), std::greater<int>());
  for (int x : sorted) { /* Process */ }
}
```

### 示例：模板特化消除运行时分支

```cpp
// Before: Runtime branch in hot loop
template<typename T>
void transform(T* data, size_t n, bool negate) {
  for (size_t i = 0; i < n; ++i) {
    if (negate) {
      data[i] = -data[i];
    } else {
      data[i] = std::abs(data[i]);
    }
  }
}

// After: Compile-time dispatch
template<bool Negate>
void transform_impl(float* data, size_t n) {
  for (size_t i = 0; i < n; ++i) {
    if constexpr (Negate) {
      data[i] = -data[i];
    } else {
      data[i] = std::abs(data[i]);
    }
  }
}

void transform(float* data, size_t n, bool negate) {
  if (negate) {
    transform_impl<true>(data, n);
  } else {
    transform_impl<false>(data, n);
  }
}
```

---

## 权衡考虑

| 技术 | 优点 | 代价 |
|------|------|------|
| Algebraic Identities | 减少昂贵操作 | 数值精度问题、可读性 |
| Short-Circuiting | 避免不必要计算 | 改变语义（副作用） |
| Reordering Tests | 减少平均检查次数 | 依赖访问模式 |
| Precomputed Tables | O(1) 查找 | 内存占用、初始化 |
| Boolean Elimination | 消除分支 | 代码重复 |

**最佳实践**：
1. Profile 确认逻辑是热点
2. 了解数据分布和访问模式
3. 注意数值精度和溢出问题
4. 保持代码可维护性
