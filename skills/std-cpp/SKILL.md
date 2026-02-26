---
name: std-cpp
description: 提供 C++ 编码规范（基于 Google C++ Style Guide）。当编写或 review C++ 代码（.cc/.cpp/.h 文件）时使用。
---

# C++ 编码规范

> 基于 Google C++ Style Guide，使用 C++17。

## 核心规则速查

| 类别 | 必须遵守 |
|------|----------|
| **版本** | C++17，不用 C++2a 功能 |
| **头文件** | self-contained，`#pragma once`，include 顺序规范 |
| **命名空间** | 必须使用，禁止头文件全局 `using` |
| **类** | 构造函数禁止调用虚函数，单参数构造函数加 `explicit` |
| **命名** | 类型 `PascalCase`，变量 `snake_case`，常量 `kCamelCase` |
| **转换** | 使用 C++ 风格（`static_cast` 等），禁止 C 风格 |
| **自增** | 迭代器用前置 `++i` |

## 完整规范

详见 [reference/full-standards.md](reference/full-standards.md)，包含：
- 规范等级定义（必须/推荐/可选）
- 头文件、作用域、类、函数规范
- 命名、注释、格式规范
