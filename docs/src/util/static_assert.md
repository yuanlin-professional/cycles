# static_assert.h - 静态断言辅助宏

## 概述

`static_assert.h` 提供编译期静态断言的辅助宏，主要用于验证数据结构的内存对齐约束。在 CUDA CUBIN 编译环境下禁用标准 `static_assert` 以避免兼容性问题。

## 类与结构体

无类或结构体定义。

## 核心函数/宏定义

| 宏 | 说明 |
|----|------|
| `static_assert(statement, message)` | 仅在 `CYCLES_CUBIN_CC` 环境下定义为空操作，覆盖标准 `static_assert`。其他环境使用 C++ 标准版本 |
| `static_assert_align(st, align)` | 检查结构体 `st` 的大小是否为 `align` 的整数倍。若不满足，编译期报错 "Structure must be strictly aligned" |

### `static_assert_align` 展开

```cpp
static_assert((sizeof(st) % (align) == 0), "Structure must be strictly aligned")
```

此宏用于确保内核数据结构满足 GPU 和 SIMD 的对齐要求。

## 依赖关系

- **内部头文件**: 无
- **外部依赖**: 无（使用 C++ 内置 `static_assert`）
- **被引用**: `kernel/types.h`

## 实现细节

1. **CUBIN 编译器兼容**：NVIDIA CUBIN 编译器（`CYCLES_CUBIN_CC`）可能不完全支持 C++11 `static_assert`，因此在该环境下将其定义为空。这不影响类型安全，因为在其他编译路径中仍会进行校验。

2. **clang-format 禁用**：文件使用 `/* clang-format off */` 标记，因为 `#define static_assert` 会触发格式化工具的解析错误。

3. **NOLINT 标记**：`static_assert_align` 宏末尾的 `// NOLINT` 抑制静态分析工具对宏中使用 `static_assert` 的警告。

4. **用途场景**：主要在 `kernel/types.h` 中用于验证传递给 GPU 的结构体对齐，例如确保 16 字节对齐以满足 float4 的 SIMD 加载要求。

## 关联文件

- `kernel/types.h` - 内核数据结构定义，使用 `static_assert_align` 验证对齐
