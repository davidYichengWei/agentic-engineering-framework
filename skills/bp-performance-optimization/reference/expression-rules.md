# Expression Rules

表达式优化技术。

---

## 1. Compile-Time Initialization

编译期初始化变量。

### 示例：constexpr 查找表

```cpp
// Before: Runtime initialization
int factorial_table[13];

void init() {
  factorial_table[0] = 1;
  for (int i = 1; i < 13; ++i) {
    factorial_table[i] = factorial_table[i-1] * i;
  }
}

// After: Compile-time initialization
constexpr int factorial(int n) {
  return n <= 1 ? 1 : n * factorial(n - 1);
}

constexpr std::array<int, 13> make_factorial_table() {
  std::array<int, 13> table{};
  for (int i = 0; i < 13; ++i) {
    table[i] = factorial(i);
  }
  return table;
}

constexpr auto factorial_table = make_factorial_table();
```

### 示例：Static 常量

```cpp
// Before: Computed at runtime
double get_pi() {
  return std::acos(-1.0);
}

// After: Compile-time constant
constexpr double kPi = 3.14159265358979323846;

// Or for complex constants
static const double kSqrt2 = std::sqrt(2.0);  // Computed once at startup
```

---

## 2. Strength Reduction

用廉价操作替代昂贵操作。

### 示例：乘法转移位

```cpp
// Before: Multiplication
int multiply_by_8(int x) { return x * 8; }
int divide_by_4(int x) { return x / 4; }

// After: Shift (for powers of 2)
int multiply_by_8(int x) { return x << 3; }
int divide_by_4(int x) { return x >> 2; }  // Note: for unsigned or positive

// Note: Modern compilers do this automatically
```

### 示例：乘法转加法（循环内）

```cpp
// Before: Multiply index every iteration
void fill_multiples(int* arr, int n, int factor) {
  for (int i = 0; i < n; ++i) {
    arr[i] = i * factor;  // Multiplication each iteration
  }
}

// After: Incremental addition
void fill_multiples(int* arr, int n, int factor) {
  int value = 0;
  for (int i = 0; i < n; ++i) {
    arr[i] = value;
    value += factor;  // Addition instead of multiplication
  }
}
```

### 示例：除法转乘法

```cpp
// Before: Division (expensive)
bool is_divisible_by_3(int x) {
  return x % 3 == 0;
}

// After: Magic number multiplication (compiler optimization)
// For constant divisors, compiler generates:
// x - (x * magic >> shift) * divisor == 0
// This is faster than hardware division

// Manual optimization for specific cases:
bool is_divisible_by_3(unsigned x) {
  // Sum of digits divisible by 3
  // Or use multiply-and-shift trick
  return (x * 0xAAAAAAAB) <= 0x55555555;
}
```

---

## 3. Common Subexpression Elimination (CSE)

消除公共子表达式。

### 示例：手动 CSE

```cpp
// Before: Repeated computation
double compute_energy(double mass, double velocity) {
  double kinetic = 0.5 * mass * velocity * velocity;
  double potential = mass * 9.81 * height;
  double total = 0.5 * mass * velocity * velocity + mass * 9.81 * height;
  return total;
}

// After: Store common subexpressions
double compute_energy(double mass, double velocity) {
  double v_squared = velocity * velocity;
  double kinetic = 0.5 * mass * v_squared;
  double mg = mass * 9.81;
  double potential = mg * height;
  double total = kinetic + potential;
  return total;
}
```

### 示例：数组索引

```cpp
// Before: Repeated index calculation
void process(int** matrix, int row, int col) {
  matrix[row * width + col] = 0;
  matrix[row * width + col + 1] = 1;
  matrix[row * width + col - 1] = 2;
}

// After: Compute base once
void process(int** matrix, int row, int col) {
  int* row_ptr = &matrix[row * width];
  row_ptr[col] = 0;
  row_ptr[col + 1] = 1;
  row_ptr[col - 1] = 2;
}
```

---

## 4. Pairing Computation

配对计算相关表达式。

### 示例：Sine/Cosine 同时计算

```cpp
// Before: Two trigonometric functions
void rotate(double angle, double* cos_out, double* sin_out) {
  *cos_out = std::cos(angle);
  *sin_out = std::sin(angle);  // Second trig call
}

// After: Compute both at once (many CPUs have fsincos instruction)
void rotate(double angle, double* cos_out, double* sin_out) {
  // sincos computes both in one operation
  sincos(angle, sin_out, cos_out);
}

// Or use complex exponential
void rotate(double angle, double* cos_out, double* sin_out) {
  std::complex<double> result = std::exp(std::complex<double>(0, angle));
  *cos_out = result.real();
  *sin_out = result.imag();
}
```

### 示例：Quotient and Remainder

```cpp
// Before: Two operations
int quotient = a / b;
int remainder = a % b;

// After: Single operation (div instruction produces both)
std::div_t result = std::div(a, b);
int quotient = result.quot;
int remainder = result.rem;

// C++11
auto [quotient, remainder] = std::div(a, b);
```

---

## 5. Word Parallelism

利用字宽并行。

### 示例：位图操作

```cpp
// Before: Byte-by-byte processing
bool all_zeros(const uint8_t* data, size_t n) {
  for (size_t i = 0; i < n; ++i) {
    if (data[i] != 0) return false;
  }
  return true;
}

// After: Word-at-a-time processing
bool all_zeros(const uint8_t* data, size_t n) {
  const uint64_t* ptr64 = reinterpret_cast<const uint64_t*>(data);
  size_t n64 = n / 8;
  
  // Check 8 bytes at a time
  for (size_t i = 0; i < n64; ++i) {
    if (ptr64[i] != 0) return false;
  }
  
  // Check remaining bytes
  for (size_t i = n64 * 8; i < n; ++i) {
    if (data[i] != 0) return false;
  }
  return true;
}
```

### 示例：Bitset 并行操作

```cpp
// Before: Separate boolean array
bool flags[1000];
int count_set() {
  int count = 0;
  for (int i = 0; i < 1000; ++i) {
    if (flags[i]) ++count;
  }
  return count;
}

// After: Bitset with popcount
std::bitset<1000> flags;
int count_set() {
  return flags.count();  // Uses hardware popcount
}
```

### 示例：SWAR (SIMD Within A Register)

```cpp
// Count bytes equal to zero in a 64-bit word
// Uses parallel byte comparison
int count_zero_bytes(uint64_t x) {
  // Magic constants for SWAR
  constexpr uint64_t lo = 0x0101010101010101ULL;
  constexpr uint64_t hi = 0x8080808080808080ULL;
  
  // Find bytes that are zero
  uint64_t zero_bytes = ~x & (x - lo) & hi;
  
  // Count number of set high bits (one per zero byte)
  return __builtin_popcountll(zero_bytes);
}
```

---

## 权衡考虑

| 技术 | 优点 | 代价 |
|------|------|------|
| Compile-Time Init | 零运行时开销 | 编译时间、constexpr 限制 |
| Strength Reduction | 减少指令周期 | 可读性、编译器通常已做 |
| CSE | 减少重复计算 | 寄存器压力 |
| Pairing | 利用硬件特性 | 平台依赖 |
| Word Parallelism | 8x/4x 吞吐 | 对齐要求、代码复杂 |

**最佳实践**：
1. 大多数 strength reduction 让编译器做
2. CSE 提高可读性的同时往往也提高性能
3. 对齐数据以启用 word parallelism
4. 使用 `constexpr` 标记编译期可计算的值
