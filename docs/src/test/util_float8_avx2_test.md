# util_float8_avx2_test.cpp - float8 类型 AVX2 指令集测试

## 概述

此文件测试 Cycles 工具库中 `vfloat8`（8 元素浮点向量）类型在 AVX2 指令集下的运算正确性。该文件通过定义 `__KERNEL_SSE__`、`__KERNEL_AVX__` 和 `__KERNEL_AVX2__` 宏来启用 AVX2 代码路径，然后包含共享的测试头文件 `util_float8_test.h` 来执行实际测试。

## 编译条件

仅在以下条件下编译测试代码：
- 目标架构为 x86/x86_64（通过 `i386`、`_M_IX86`、`__x86_64__`、`_M_X64` 预处理器宏检测）
- 编译器支持 AVX2 指令集（`__AVX2__` 已定义）
- 非 Apple 平台（在 CMakeLists.txt 中排除）

## 宏定义

| 宏名称 | 值 | 说明 |
|--------|------|------|
| `__KERNEL_SSE__` | (已定义) | 启用 SSE 代码路径 |
| `__KERNEL_AVX__` | (已定义) | 启用 AVX 代码路径 |
| `__KERNEL_AVX2__` | (已定义) | 启用 AVX2 代码路径 |
| `TEST_CATEGORY_NAME` | `util_avx2` | 测试类别名称，用于 GTest 中区分不同指令集的测试 |

## 测试用例

所有测试用例定义在 `util_float8_test.h` 中（详见 `util_float8_test.md`），以 `util_avx2` 作为测试类别名称运行。

## 依赖关系
- **被测源文件**: `util/types.h`（vfloat8 类型定义）
- **测试框架**: Google Test (GTest)
- **共享测试头**: `util_float8_test.h`

## 关联文件
- `src/test/util_float8_test.h` - 共享的 float8 测试用例定义
- `src/test/util_float8_avx_test.cpp` - AVX 版本测试
- `src/test/util_float8_sse2_test.cpp` - SSE2 版本测试
