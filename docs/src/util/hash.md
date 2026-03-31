# hash.h - GPU/CPU 通用哈希函数集合

## 概述

`hash.h` 是 Cycles 渲染器中最核心的哈希函数库，提供了多种高质量哈希算法的实现，用于噪声生成、采样、着色器计算等场景。所有函数均使用 `ccl_device_inline` 或 `ccl_device_forceinline` 修饰，可同时在 CPU 和 GPU（CUDA/HIP/Metal/OptiX）上运行。文件中包含 PCG 哈希、Jenkins Lookup3 哈希、Hash Prospector 哈希、Modified Wang 哈希以及索引洗牌哈希等多种算法。

## 类与结构体

该文件不定义类或结构体，全部为内联函数和宏定义。

## 核心函数

### 整数到浮点数转换

- `float uint_to_float_excl(const uint n)` -- 将 `[0, UINT_MAX]` 映射到 `[0.0, 1.0)`（开区间），除以 4294967808 而非 2^32 以避免浮点舍入导致的边界问题
- `float uint_to_float_incl(const uint n)` -- 将 `[0, UINT_MAX]` 映射到 `[0.0, 1.0]`（闭区间）

### PCG 哈希族（2D/3D/4D）

基于论文 "Hash Functions for GPU Rendering"（JCGT 2020）实现，使用有符号整数以兼容 OSL。

- `int2 hash_pcg2d_i(int2 v)` -- 2D PCG 哈希，输出范围 `[0, 0x7FFFFFFF]`
- `int3 hash_pcg3d_i(int3 v)` -- 3D PCG 哈希
- `int4 hash_pcg4d_i(int4 v)` -- 4D PCG 哈希

### Jenkins Lookup3 哈希族

基于 Bob Jenkins 的 Lookup3 算法，通过宏 `rot`、`mix`、`final` 实现混合操作。

- `uint hash_uint(const uint kx)` -- 单 uint 哈希
- `uint hash_uint2(const uint kx, const uint ky)` -- 双 uint 哈希
- `uint hash_uint3(const uint kx, const uint ky, const uint kz)` -- 三 uint 哈希
- `uint hash_uint4(const uint kx, const uint ky, const uint kz, const uint kw)` -- 四 uint 哈希

### uint/float 到 float 的转换哈希

- `float hash_uint_to_float(const uint kx)` -- uint -> `[0, 1]` 浮点
- `float hash_uint2_to_float(...)` / `hash_uint3_to_float(...)` / `hash_uint4_to_float(...)` -- 多 uint -> 浮点
- `float hash_float_to_float(const float k)` -- float -> `[0, 1]` 浮点
- `float hash_float2_to_float(const float2 k)` -- float2 -> 浮点
- `float hash_float3_to_float(const float3 k)` / `hash_float4_to_float(const float4 k)` -- 多维 float -> 浮点

### 多维向量哈希（int -> float 向量，基于 PCG）

- `float2 hash_int2_to_float2(const int2 k)` -- int2 -> float2
- `float3 hash_int3_to_float3(const int3 k)` -- int3 -> float3
- `float4 hash_int4_to_float4(const int4 k)` -- int4 -> float4
- `float3 hash_int2_to_float3(const int2 k)` / `float3 hash_int4_to_float3(const int4 k)` -- 维度转换变体

### 多维向量哈希（float -> float 向量）

- `float2 hash_float2_to_float2(const float2 k)` -- float2 -> float2
- `float3 hash_float3_to_float3(const float3 k)` -- float3 -> float3
- `float4 hash_float4_to_float4(const float4 k)` -- float4 -> float4
- 以及各种维度交叉变体：`hash_float_to_float2`, `hash_float_to_float3`, `hash_float2_to_float3`, `hash_float3_to_float2`, `hash_float4_to_float2`, `hash_float4_to_float3`

### SSE/AVX 优化版 Jenkins Lookup3

在 `__KERNEL_SSE__` 宏下提供 SIMD 批量哈希版本：

- `int4 hash_int4(...)` / `hash_int4_2(...)` / `hash_int4_3(...)` / `hash_int4_4(...)` -- SSE int4 批量哈希
- `vint8 hash_int8(...)` / `hash_int8_2(...)` / `hash_int8_3(...)` / `hash_int8_4(...)` -- AVX2 vint8 批量哈希（在 `__KERNEL_AVX2__` 下启用）

### Hash Prospector 哈希

基于 https://github.com/skeeto/hash-prospector 的高质量 32 位哈希。

- `uint hash_hp_uint(uint i)` -- 单 uint 哈希（输入 0 不映射为 0，额外做了 XOR）
- `uint hash_hp_seeded_uint(const uint i, uint seed)` -- 带种子版本
- `float hash_hp_float(const uint i)` -- 输出 `[0.0, 1.0)`
- `float hash_hp_seeded_float(const uint i, const uint seed)` -- 带种子浮点版本

### Modified Wang 哈希

基于 Wang Hash 的定制变体，速度更快但质量略低。

- `uint hash_wang_seeded_uint(uint i, const uint seed)` -- 带种子 uint 哈希
- `float hash_wang_seeded_float(const uint i, const uint seed)` -- 带种子浮点输出 `[0.0, 1.0)`

### 索引洗牌函数

- `uint hash_shuffle_uint(uint i, const uint length, const uint seed)` -- 将索引 `i` 在长度 `length` 的范围内进行随机洗牌，平均 O(1) 时间复杂度，无需实际打乱数组，适用于随机顺序遍历

### 2D 哈希（IQ 方法）

- `uint hash_iqnt2d(const uint x, const uint y)` -- 基于 JCGT 推荐的 2D 哈希

### 字符串哈希

- `uint hash_string(const char *str)` -- 简单的字符串哈希（仅 CPU 端，`__KERNEL_GPU__` 下不可用），使用乘法散列法

## 依赖关系

- **内部头文件**: `util/defines.h`, `util/math.h`, `util/types.h`
- **被引用**: `kernel/svm/voronoi.h`, `kernel/svm/white_noise.h`, `kernel/svm/noise.h`, `kernel/svm/geometry.h`, `kernel/svm/gabor.h`, `kernel/sample/lcg.h`, `kernel/sample/pattern.h`, `kernel/sample/sobol_burley.h`, `kernel/sample/tabulated_sobol.h`, `kernel/osl/services_gpu.h`, `scene/tabulated_sobol.cpp`, `scene/volume.cpp`, `scene/integrator.cpp`, `scene/light.cpp`, `subd/subpatch.h`, `hydra/geometry.inl`, `hydra/light.cpp`, `app/cycles_precompute.cpp`

## 关联文件

- `util/murmurhash.h` -- MurmurHash3 哈希实现，用于 Cryptomatte
- `util/md5.h` -- MD5 哈希实现，用于磁盘缓存
- `util/types.h` -- 提供 `int2`, `int3`, `int4`, `float2`, `float3`, `float4` 等向量类型定义
