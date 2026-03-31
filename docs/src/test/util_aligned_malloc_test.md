# util_aligned_malloc_test.cpp - 对齐内存分配测试

## 概述

此文件测试 Cycles 工具库中的对齐内存分配函数 `util_aligned_malloc` 和 `util_aligned_free`。对齐内存分配对于 SIMD 指令优化至关重要，因为 SSE/AVX 等指令集要求操作数地址按特定字节数对齐。

## 测试用例

### TEST(util_aligned_malloc, aligned_malloc_16)
- **功能**: 验证 16 字节对齐的内存分配。分配一个 `int` 大小的内存块，检查返回的指针地址是否为 16 的整数倍，然后释放内存。
- **平台**: 所有平台均执行

### TEST(util_aligned_malloc, aligned_malloc_32)
- **功能**: 验证 32 字节对齐的内存分配。分配一个 `int` 大小的内存块，检查返回的指针地址是否为 32 的整数倍，然后释放内存。
- **平台**: 仅非 Apple 平台执行（Apple 平台当前仅支持 16 字节对齐）

## 辅助宏

| 宏名称 | 说明 |
|--------|------|
| `CHECK_ALIGNMENT(ptr, align)` | 检查指针 `ptr` 的地址是否按 `align` 字节对齐 |

## 依赖关系
- **被测源文件**: `util/aligned_malloc.h`
- **测试框架**: Google Test (GTest)

## 关联文件
- `src/util/aligned_malloc.h` - 对齐内存分配函数声明
- `src/util/aligned_malloc.cpp` - 对齐内存分配函数实现
