# Space-for-Time Rules

用空间换取时间的优化技术。

---

## 1. Data Structure Augmentation

添加冗余信息以加速操作。

### 示例：链表添加 size 字段

```cpp
// Before: O(n) to get size
struct LinkedList {
  Node* head;
  
  size_t size() const {
    size_t count = 0;
    for (Node* n = head; n; n = n->next) count++;
    return count;
  }
};

// After: O(1) to get size
struct LinkedList {
  Node* head;
  size_t size_;  // Augmented field
  
  size_t size() const { return size_; }
  
  void insert(Node* node) {
    node->next = head;
    head = node;
    ++size_;  // Maintain augmented field
  }
};
```

### 示例：树节点添加 parent 指针

```cpp
// Before: Finding parent requires traversal from root
struct TreeNode {
  int value;
  TreeNode* left;
  TreeNode* right;
};

// After: O(1) parent access
struct TreeNode {
  int value;
  TreeNode* left;
  TreeNode* right;
  TreeNode* parent;  // Augmented field
};
```

---

## 2. Store Precomputed Results

预计算并存储结果，避免重复计算。

### 示例：预计算阶乘表

```cpp
// Before: Compute factorial every time
uint64_t factorial(int n) {
  uint64_t result = 1;
  for (int i = 2; i <= n; ++i) {
    result *= i;
  }
  return result;
}

// After: Table lookup
constexpr int kMaxFactorial = 20;
uint64_t factorial_table[kMaxFactorial + 1];

void init_factorial_table() {
  factorial_table[0] = 1;
  for (int i = 1; i <= kMaxFactorial; ++i) {
    factorial_table[i] = factorial_table[i - 1] * i;
  }
}

uint64_t factorial(int n) {
  return factorial_table[n];  // O(1)
}
```

### 示例：预计算 CRC 查找表

```cpp
// Before: Bit-by-bit CRC computation
uint32_t crc32_slow(const uint8_t* data, size_t len) {
  uint32_t crc = 0xFFFFFFFF;
  for (size_t i = 0; i < len; ++i) {
    crc ^= data[i];
    for (int j = 0; j < 8; ++j) {
      crc = (crc >> 1) ^ ((crc & 1) ? 0xEDB88320 : 0);
    }
  }
  return ~crc;
}

// After: Table-driven CRC (8x fewer iterations in inner loop)
uint32_t crc_table[256];

void init_crc_table() {
  for (uint32_t i = 0; i < 256; ++i) {
    uint32_t crc = i;
    for (int j = 0; j < 8; ++j) {
      crc = (crc >> 1) ^ ((crc & 1) ? 0xEDB88320 : 0);
    }
    crc_table[i] = crc;
  }
}

uint32_t crc32_fast(const uint8_t* data, size_t len) {
  uint32_t crc = 0xFFFFFFFF;
  for (size_t i = 0; i < len; ++i) {
    crc = (crc >> 8) ^ crc_table[(crc ^ data[i]) & 0xFF];
  }
  return ~crc;
}
```

---

## 3. Caching

缓存频繁访问的数据。

### 示例：LRU Cache

```cpp
// Expensive function
std::string fetch_user_profile(int user_id) {
  // Slow database query...
  return db_query("SELECT * FROM users WHERE id = ?", user_id);
}

// With caching
class ProfileCache {
  std::unordered_map<int, std::string> cache_;
  std::list<int> lru_order_;
  static constexpr size_t kMaxSize = 1000;

 public:
  std::string get(int user_id) {
    auto it = cache_.find(user_id);
    if (it != cache_.end()) {
      // Move to front (most recently used)
      touch(user_id);
      return it->second;
    }
    
    // Cache miss: fetch and store
    std::string profile = fetch_user_profile(user_id);
    put(user_id, profile);
    return profile;
  }

 private:
  void put(int key, const std::string& value) {
    if (cache_.size() >= kMaxSize) {
      evict_lru();
    }
    cache_[key] = value;
    lru_order_.push_front(key);
  }
  
  void touch(int key) { /* move key to front of lru_order_ */ }
  void evict_lru() { /* remove least recently used */ }
};
```

### 示例：Memoization

```cpp
// Before: Exponential time fibonacci
int fib(int n) {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2);
}

// After: Linear time with memoization
std::unordered_map<int, int> fib_cache;

int fib_memo(int n) {
  if (n <= 1) return n;
  
  auto it = fib_cache.find(n);
  if (it != fib_cache.end()) {
    return it->second;
  }
  
  int result = fib_memo(n - 1) + fib_memo(n - 2);
  fib_cache[n] = result;
  return result;
}
```

---

## 4. Lazy Evaluation

延迟计算直到真正需要。

### 示例：惰性初始化

```cpp
// Before: Always compute even if unused
class Config {
 public:
  Config() {
    expensive_data_ = compute_expensive_data();  // Always runs
  }
  
  Data& data() { return expensive_data_; }

 private:
  Data expensive_data_;
};

// After: Compute only when needed
class Config {
 public:
  Data& data() {
    if (!data_initialized_) {
      expensive_data_ = compute_expensive_data();
      data_initialized_ = true;
    }
    return expensive_data_;
  }

 private:
  Data expensive_data_;
  bool data_initialized_ = false;
};

// C++11+ version using std::optional or call_once
class Config {
 public:
  Data& data() {
    std::call_once(init_flag_, [this]() {
      expensive_data_ = compute_expensive_data();
    });
    return expensive_data_;
  }

 private:
  Data expensive_data_;
  std::once_flag init_flag_;
};
```

### 示例：Copy-on-Write (COW)

```cpp
// Lazy copy: share data until modification
class CowString {
 public:
  CowString(const char* s) : data_(std::make_shared<std::string>(s)) {}
  
  // Copy constructor: just share the pointer (cheap)
  CowString(const CowString& other) : data_(other.data_) {}
  
  // Modification: copy if shared
  void set(size_t pos, char c) {
    if (data_.use_count() > 1) {
      // Make a private copy before modification
      data_ = std::make_shared<std::string>(*data_);
    }
    (*data_)[pos] = c;
  }
  
  const std::string& str() const { return *data_; }

 private:
  std::shared_ptr<std::string> data_;
};
```

---

## 权衡考虑

| 技术 | 优点 | 代价 |
|------|------|------|
| Augmentation | 快速查询 | 维护开销、一致性风险 |
| Precomputation | 极快查找 | 初始化时间、内存占用 |
| Caching | 加速重复访问 | 内存占用、缓存失效逻辑 |
| Lazy Evaluation | 避免不必要计算 | 首次访问延迟、线程安全 |

**何时使用**：
- 读多写少的场景
- 计算成本远高于内存访问成本
- 存在明显的访问局部性
