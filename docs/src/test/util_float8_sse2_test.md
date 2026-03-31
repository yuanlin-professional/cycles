# util_float8_sse2_test.cpp - float8 类型 SSE2 指令集测试

## 概述

此文件测试 Cycles 工具库中 `vfloat8`（8 元素浮点向量）类型在 SSE2 指令集下的运算正确性。SSE2 是最基础的 SIMD 指令集（仅支持 128 位操作），此测试验证 `vfloat8` 在没有原生 256 位支持时通过两个 128 位操作模拟的正确性。

## 编译条件

仅在以下条件下编译测试代码：
- 目标架构为 x86/x86_64
- 编译器支持 SSE2 指令集（`__SSE2__` 已定义）

## 宏定义

| 宏名称 | 值 | 说明 |
|--------|------|------|
| `__KERNEL_SSE__` | (已定义) | 启用 SSE 代码路径 |
| `__KERNEL_SSE2__` | (已定义) | 启用 SSE2 代码路径 |
| `TEST_CATEGORY_NAME` | `util_sse2` | 测试类别名称 |

## 测试用例

所有测试用例定义在 `util_float8_test.h` 中（详见 `util_float8_test.md`），以 `util_sse2` 作为测试类别名称运行。

## 依赖关系
- **被测源文件**: `util/types.h`（vfloat8 类型定义）
- **测试框架**: Google Test (GTest)
- **共享测试头**: `util_float8_test.h`

## 关联文件
- `src/test/util_float8_test.h` - 共享的 float8 测试用例定义
- `src/test/util_float8_avx_test.cpp` - AVX 版本测试
- `src/test/util_float8_avx2_test.cpp` - AVX2 版本测试
