# 可读性详细规则

## 格式化

使用项目的格式化工具自动格式化代码，不要手动格式化。

编辑现有代码时，遵循文件已有风格。一致性 > 个人偏好。

---

## 命名详细规则

### 描述性命名

```cpp
// ❌ 含糊
int n = 0;

// ✅ 描述性
int retry_count = 0;
```

### 避免魔法数字

```cpp
// ❌ 魔法数字
if (t > 86400 && s == 1) {
    p = t * 10;
}

// ✅ 命名常量
const int SECONDS_IN_DAY = 86400;
const int STATUS_ACTIVE = 1;

if (timeWorkedSeconds > SECONDS_IN_DAY && userStatus == STATUS_ACTIVE) {
    dailyPay = timeWorkedSeconds * HOURLY_RATE;
}
```

### 布尔命名

布尔变量命名为问题形式：`isValid`, `hasAccess`, `canProcess`

---

## 作用域

### 窄作用域

```cpp
// ❌ 宽作用域
int result;
for (int i = 0; i < 10; ++i) {
  if (auto v = compute(i)) {
    result = *v;
    break;
  }
}

// ✅ 窄作用域
for (int i = 0; i < 10; ++i) {
  if (auto v = compute(i)) {
    const int result = *v;
    use(result);
    break;
  }
}
```

### 声明时初始化

```cpp
// ❌ 先声明后赋值
int fd;
fd = open(path.c_str(), O_RDONLY);

// ✅ 声明即初始化
int fd = open(path.c_str(), O_RDONLY);
```

### 每行一个声明

```cpp
// ❌ 容易误解（q 是 int，不是 int*）
int* p, q;

// ✅ 清晰
int* p = nullptr;
int  q = 0;
```

---

## 控制流

### Guard Clause 详细示例

```cpp
// ❌ 箭头代码（深层嵌套）
void ProcessOrder(Order* order) {
    if (order != nullptr) {
        if (order->isValid()) {
            if (order->hasItems()) {
                // ... 10 lines of logic ...
            } else {
                // error handling
            }
        } else {
             // error handling
        }
    }
}

// ✅ Guard Clause
void ProcessOrder(Order* order) {
    if (order == nullptr) return;
    if (!order->isValid()) {
        Log("Invalid order");
        return;
    }
    if (!order->hasItems()) {
        Log("Order is empty");
        return;
    }

    // Main Logic (not nested!)
    // ... 10 lines of logic ...
}
```

### 返回值设计

```cpp
Status process(const Request& r) {
  if (!r.is_valid()) return Status::InvalidArgument;
  if (r.items().empty()) return Status::Ok;

  // Main path stays left-aligned.
  return do_process(r);
}
```

---

## 注释详细规则

### 解释 Why，不是 What

```cpp
// ❌ 复述代码
// Increment i by 1
++i;

// ✅ 解释意图/假设
// We intentionally skip index 0 because it is the sentinel slot.
for (size_t i = 1; i < slots.size(); ++i) { /* ... */ }
```

### TODO 格式

```cpp
// TODO(username): Refactor this to use Factory pattern once DB schema is finalized
User u;
```

---

## 使用标准库算法

优先使用命名算法而非手写循环（当意图更清晰时）：

```cpp
#include <algorithm>
#include <vector>

// ❌ 手写循环
int count = 0;
for (int i = 0; i < 100; i++) {
    if (data[i] > 50) count++;
}

// ✅ 标准算法（意图更清晰）
int count = std::count_if(data.begin(), data.end(), [](int i) {
    return i > 50;
});
```
