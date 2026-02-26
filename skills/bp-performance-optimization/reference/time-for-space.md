# Time-for-Space Rules

用时间换取空间的优化技术。

---

## 1. Packing

使用紧凑存储表示减少内存占用。

### 示例：位域压缩

```cpp
// Before: 12 bytes (with padding)
struct Flags {
  bool is_active;      // 1 byte + 3 padding
  bool is_verified;    // 1 byte + 3 padding
  bool is_admin;       // 1 byte + 3 padding
};

// After: 1 byte
struct Flags {
  uint8_t is_active   : 1;
  uint8_t is_verified : 1;
  uint8_t is_admin    : 1;
  uint8_t reserved    : 5;
};

// Or use bitmask
class Flags {
 public:
  static constexpr uint8_t kActive   = 1 << 0;
  static constexpr uint8_t kVerified = 1 << 1;
  static constexpr uint8_t kAdmin    = 1 << 2;
  
  bool is_active() const { return flags_ & kActive; }
  void set_active(bool v) { 
    flags_ = v ? (flags_ | kActive) : (flags_ & ~kActive); 
  }

 private:
  uint8_t flags_ = 0;
};
```

### 示例：变长编码 (Varint)

```cpp
// Before: Fixed 8 bytes per integer
void write_fixed(uint64_t value, std::vector<uint8_t>& out) {
  for (int i = 0; i < 8; ++i) {
    out.push_back(value & 0xFF);
    value >>= 8;
  }
}

// After: 1-10 bytes depending on value (smaller values use less space)
void write_varint(uint64_t value, std::vector<uint8_t>& out) {
  while (value >= 0x80) {
    out.push_back((value & 0x7F) | 0x80);
    value >>= 7;
  }
  out.push_back(value);
}

uint64_t read_varint(const uint8_t*& data) {
  uint64_t result = 0;
  int shift = 0;
  while (*data & 0x80) {
    result |= static_cast<uint64_t>(*data & 0x7F) << shift;
    ++data;
    shift += 7;
  }
  result |= static_cast<uint64_t>(*data) << shift;
  ++data;
  return result;
}
```

### 示例：结构体重排减少 padding

```cpp
// Before: 24 bytes (with padding)
struct BadLayout {
  char a;        // 1 byte + 7 padding
  double b;      // 8 bytes
  char c;        // 1 byte + 7 padding
};

// After: 16 bytes (optimal)
struct GoodLayout {
  double b;      // 8 bytes
  char a;        // 1 byte
  char c;        // 1 byte + 6 padding
};

// Rule: Sort members by size, largest first
```

---

## 2. Overlaying

在同一内存空间存储不会同时使用的数据。

### 示例：Union

```cpp
// Before: Separate storage for each type
struct Value {
  ValueType type;
  int64_t int_value;      // 8 bytes
  double float_value;     // 8 bytes
  std::string str_value;  // 32 bytes
  // Total: 48+ bytes
};

// After: Overlay with union (or std::variant)
struct Value {
  ValueType type;
  union {
    int64_t int_value;
    double float_value;
    // Note: non-trivial types like std::string need special handling
  } data;
  // For strings, store separately or use std::variant
};

// Modern C++: std::variant
using Value = std::variant<int64_t, double, std::string>;
```

### 示例：Memory Pool with Reuse

```cpp
// Reuse memory for objects that have non-overlapping lifetimes
class ObjectPool {
 public:
  void* allocate(size_t size) {
    if (!free_list_.empty() && free_list_.back().size >= size) {
      void* ptr = free_list_.back().ptr;
      free_list_.pop_back();
      return ptr;
    }
    return malloc(size);
  }
  
  void deallocate(void* ptr, size_t size) {
    // Don't free, add to free list for reuse
    free_list_.push_back({ptr, size});
  }

 private:
  struct Block { void* ptr; size_t size; };
  std::vector<Block> free_list_;
};
```

---

## 3. Interpreters

用解释器紧凑表示程序或数据。

### 示例：字节码替代函数调用

```cpp
// Before: Each operation is a function call
void process_commands(const std::vector<Command>& commands) {
  for (const auto& cmd : commands) {
    switch (cmd.type) {
      case ADD: add(cmd.a, cmd.b); break;
      case SUB: sub(cmd.a, cmd.b); break;
      case MUL: mul(cmd.a, cmd.b); break;
      // ... many more
    }
  }
}

// After: Compact bytecode representation
enum OpCode : uint8_t { OP_ADD, OP_SUB, OP_MUL, OP_HALT };

void execute_bytecode(const uint8_t* code) {
  int stack[256];
  int sp = 0;
  
  while (true) {
    switch (*code++) {
      case OP_ADD:
        stack[sp - 2] = stack[sp - 2] + stack[sp - 1];
        --sp;
        break;
      case OP_SUB:
        stack[sp - 2] = stack[sp - 2] - stack[sp - 1];
        --sp;
        break;
      case OP_HALT:
        return;
    }
  }
}
```

### 示例：有限状态机压缩

```cpp
// Before: Explicit state handling with many conditionals
void parse_json_string(const char* input) {
  bool in_string = false;
  bool escaped = false;
  // Complex nested conditionals...
}

// After: Compact transition table
enum State { START, IN_STRING, ESCAPED, END };
enum Input { QUOTE, BACKSLASH, OTHER };

// Transition table: [current_state][input] -> next_state
const State transitions[4][3] = {
  /* START */     { IN_STRING, START,     START },
  /* IN_STRING */ { END,       ESCAPED,   IN_STRING },
  /* ESCAPED */   { IN_STRING, IN_STRING, IN_STRING },
  /* END */       { END,       END,       END }
};

State parse_char(State current, char c) {
  Input input = (c == '"') ? QUOTE : (c == '\\') ? BACKSLASH : OTHER;
  return transitions[current][input];
}
```

### 示例：Run-Length Encoding

```cpp
// Before: Store each pixel
std::vector<uint8_t> image;  // 1 byte per pixel

// After: Store (count, value) pairs for runs
struct RLEEntry {
  uint8_t count;  // Up to 255 consecutive same values
  uint8_t value;
};
std::vector<RLEEntry> compressed_image;

// Compression
std::vector<RLEEntry> rle_encode(const std::vector<uint8_t>& data) {
  std::vector<RLEEntry> result;
  size_t i = 0;
  while (i < data.size()) {
    uint8_t value = data[i];
    uint8_t count = 1;
    while (i + count < data.size() && 
           data[i + count] == value && 
           count < 255) {
      ++count;
    }
    result.push_back({count, value});
    i += count;
  }
  return result;
}
```

---

## 权衡考虑

| 技术 | 优点 | 代价 |
|------|------|------|
| Packing | 减少内存占用 | 访问开销、代码复杂度 |
| Overlaying | 内存复用 | 生命周期管理复杂 |
| Interpreters | 紧凑表示 | 执行开销、调试困难 |

**何时使用**：
- 内存是瓶颈（嵌入式、大数据集）
- 需要序列化/网络传输
- 数据访问频率低于内存节省收益
- 数据有明显的冗余模式
