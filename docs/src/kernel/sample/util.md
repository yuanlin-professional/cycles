# util.h - 采样工具函数（Owen 扰乱与 Morton 编码）

## 概述
本文件提供采样子系统所需的底层工具函数，包括 Owen 扰乱（base-2 和 base-4）以及 Morton 曲线编码。这些函数是 Sobol-Burley 采样器和蓝噪声采样器的核心构件。Owen 扰乱用于对低差异序列进行分层随机置换以打破结构化模式，Morton 编码用于将二维像素坐标映射为一维索引以支持蓝噪声采样的空间分层。

## 类与结构体
无。

## 枚举与常量
无。

## 核心函数

### reversed_bit_owen()
- **签名**: `ccl_device_inline uint reversed_bit_owen(uint n, const uint seed)`
- **功能**: 对位反转的无符号整数执行 base-2 Owen 扰乱。等价于 Laine-Karras 置换，但质量更高。该函数通过一系列乘法、加法和异或操作实现，是基于哈希的 Owen 扰乱的核心。
  - 输入 `n` 必须是位反转后的值
  - `seed` 控制扰乱模式
  - 算法来源：https://psychopath.io/post/2021_01_30_building_a_better_lk_hash

### reversed_bit_owen_base4()
- **签名**: `ccl_device_inline uint reversed_bit_owen_base4(uint n, const uint seed)`
- **功能**: 对位反转的无符号整数执行 base-4 Owen 扰乱。与 base-2 版本类似，但额外使用位对交互操作（`(n >> 1) & (n << 1) & 0x55555555`）来处理 base-4 数位之间的依赖关系。
  - 算法来源：https://psychopath.io/post/2022_08_14_a_fast_hash_for_base_4_owen_scrambling

### nested_uniform_scramble()
- **签名**: `ccl_device_inline uint nested_uniform_scramble(const uint i, const uint seed)`
- **功能**: 对普通（非位反转）无符号整数执行 base-2 Owen 扰乱。内部先将输入位反转，调用 `reversed_bit_owen`，再将结果位反转回来。这是面向上层调用的便捷包装。

### nested_uniform_scramble_base4()
- **签名**: `ccl_device_inline uint nested_uniform_scramble_base4(const uint i, const uint seed)`
- **功能**: 对普通无符号整数执行 base-4 Owen 扰乱。`reversed_bit_owen_base4` 的便捷包装，处理位反转的输入/输出转换。

### expand_bits()
- **签名**: `ccl_device_inline uint expand_bits(uint x)`
- **功能**: 将 16 位整数的位展开为 32 位，每个原始位之间插入一个零位。例如 `0b1010` 变为 `0b01000100`。这是 Morton 编码的基础操作，通过四步位移-掩码操作高效实现。

### morton2d()
- **签名**: `ccl_device_inline uint morton2d(const uint x, const uint y)`
- **功能**: 将二维坐标 `(x, y)` 编码为一维 Morton 码（Z 阶曲线索引）。通过交错 x 和 y 的展开位实现：x 占偶数位，y 占奇数位。Morton 码保持空间局部性，是蓝噪声采样器中像素序列排序的基础。

## 依赖关系
- **内部头文件**: `util/math.h`、`util/types.h`
- **被引用**:
  - `src/kernel/sample/sobol_burley.h` - 调用 `reversed_bit_owen` 进行 Sobol 序列的 Owen 扰乱
  - `src/kernel/sample/tabulated_sobol.h` - 调用 `nested_uniform_scramble` 进行样本索引扰乱
  - `src/kernel/sample/pattern.h`（间接）- 通过 `sobol_burley.h` 和 `tabulated_sobol.h` 使用

## 实现细节 / 关键算法

### Owen 扰乱原理
Owen 扰乱（也称为嵌套均匀扰乱，Nested Uniform Scrambling）是一种对低差异序列进行分层随机置换的技术。它保持了序列的低差异性（即良好的均匀覆盖性），同时消除了原始序列的确定性结构模式。

传统 Owen 扰乱需要构造一棵随机二叉/四叉树，时间和空间复杂度较高。本文件使用基于哈希的方法（Burley 2020, psychopath.io），将树遍历替换为简单的整数哈希操作，实现 O(1) 的单样本扰乱计算。

### base-2 vs base-4 Owen 扰乱
- **base-2**（`reversed_bit_owen`）：在二进制数位级别进行扰乱，每一位的置换取决于其所有更高位。用于 Sobol 序列的扰乱。
- **base-4**（`reversed_bit_owen_base4`）：在 base-4 数位级别进行扰乱，每两个二进制位组成一个 base-4 数位。额外的位对交互操作 `(n >> 1) & (n << 1) & 0x55555555` 捕获了同一 base-4 数位内两个二进制位之间的依赖关系。用于蓝噪声采样器中 Morton 索引的扰乱，因为 Morton 码天然以 base-4 组织（每个 base-4 数位对应一个二维象限）。

### Morton 码与空间填充曲线
Morton 码（Z 阶曲线）通过交错两个坐标的二进制位，将二维平面映射到一维序列。其关键性质是空间局部性：二维空间中相近的点在一维序列中也倾向于接近。

`expand_bits` 使用经典的并行位操作方法，在四步内完成 16 位到 32 位的展开：
```
x &= 0x0000ffff;           // 保留低16位
x = (x ^ (x << 8)) & 0x00ff00ff;   // 每8位分组
x = (x ^ (x << 4)) & 0x0f0f0f0f;   // 每4位分组
x = (x ^ (x << 2)) & 0x33333333;   // 每2位分组
x = (x ^ (x << 1)) & 0x55555555;   // 每1位分组（最终展开）
```

### 在采样管线中的位置
本文件处于采样子系统的最底层：
- `util.h` 提供 Owen 扰乱和 Morton 编码原语
- `sobol_burley.h` 和 `tabulated_sobol.h` 使用这些原语构建完整的采样器
- `pattern.h` 调度上层请求到具体采样器

## 关联文件
- `src/kernel/sample/sobol_burley.h` - Sobol-Burley 采样器，主要调用方
- `src/kernel/sample/tabulated_sobol.h` - 预计算表格 Sobol 采样器，主要调用方
- `src/kernel/sample/pattern.h` - 采样模式调度器，调用 `morton2d` 和 `nested_uniform_scramble_base4` 用于蓝噪声像素索引
- `src/util/math.h` - 数学工具函数
- `src/util/types.h` - 类型定义
