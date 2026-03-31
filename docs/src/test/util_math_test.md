# util_math_test.cpp - 基础数学工具函数测试

## 概述

此文件测试 Cycles 工具库中的基础数学工具函数，包括二的幂次运算和整数位反转操作。这些函数在渲染器的内存分配对齐、哈希计算和采样序列生成等场景中广泛使用。

## 测试用例

### TEST(math, next_power_of_two)
- **功能**: 验证 `next_power_of_two` 函数返回严格大于输入值的最小二的幂次。
- **验证数据**:
  - `next_power_of_two(0) == 1`
  - `next_power_of_two(1) == 2`
  - `next_power_of_two(2) == 4`
  - `next_power_of_two(3) == 4`
  - `next_power_of_two(4) == 8`

### TEST(math, prev_power_of_two)
- **功能**: 验证 `prev_power_of_two` 函数返回严格小于输入值的最大二的幂次（或对于 0 返回 0）。
- **验证数据**:
  - `prev_power_of_two(0) == 0`
  - `prev_power_of_two(1) == 1`, `prev_power_of_two(2) == 1`
  - `prev_power_of_two(3) == 2`, `prev_power_of_two(4) == 2`
  - `prev_power_of_two(5..8) == 4`

### TEST(math, reverse_integer_bits)
- **功能**: 验证 `reverse_integer_bits` 函数将 32 位整数的二进制位完全反转。
- **验证数据**:
  - `0xFFFFFFFF -> 0xFFFFFFFF`（全 1 不变）
  - `0x00000000 -> 0x00000000`（全 0 不变）
  - `0x1 -> 0x80000000`（最低位移到最高位）
  - `0x80000000 -> 0x1`（最高位移到最低位）
  - `0xFFFF0000 -> 0x0000FFFF`（高 16 位与低 16 位互换并反转）
  - `0xAAAAAAAA -> 0x55555555`（交替位模式反转）

## 依赖关系
- **被测源文件**: `util/math.h`
- **测试框架**: Google Test (GTest)

## 关联文件
- `src/util/math.h` - 数学工具函数定义
