# cycles_precompute.cpp - GGX 微表面 BSDF 预计算查找表生成工具

## 概述

`cycles_precompute.cpp` 是一个离线预计算工具，用于通过蒙特卡洛积分生成 GGX 微表面模型的能量守恒查找表（Look-Up Table）。这些预计算表在运行时被 Cycles 内核（Kernel）用于快速查询微表面 BSDF 的总反照率（albedo）和 Fresnel 插值因子，以实现能量守恒的着色器效果。生成的结果以 C++ 静态数组的形式输出到标准输出，可直接嵌入到内核源码中。

## 类与结构体

### PrecomputeTerm
- **功能**: 描述一个预计算项的参数配置
- **关键成员**:
  - `samples` (`int`) — 蒙特卡洛采样数，控制积分精度（通常为 2^20 到 2^26）
  - `nx, ny, nz` (`int`) — 查找表在三个维度上的分辨率（粗糙度、余弦入射角、IOR/指数参数）
  - `evaluation` (`std::function<float(float, float, float, float3)>`) — 评估函数，接收 (rough, mu, z, rand) 参数并返回单次采样的贡献值

## 核心函数

### precompute_ggx_E()
- **签名**: `static float precompute_ggx_E(const float rough, const float mu, const float3 rand)`
- **功能**: 计算 GGX 微表面 BRDF 在给定粗糙度和入射余弦角下的单次采样反照率。设置 IOR=1.0（无 Fresnel 效应）的 `MicrofacetBsdf`，调用 `bsdf_microfacet_ggx_sample` 进行重要性采样，返回 `eval/pdf` 作为蒙特卡洛估计量

### precompute_ggx_glass_E()
- **签名**: `static float precompute_ggx_glass_E(const float rough, const float mu, const float eta, const float3 rand)`
- **功能**: 计算带介电 Fresnel 的 GGX 微表面 BSDF（玻璃模型）在给定粗糙度、入射余弦角和折射率下的单次采样反照率。使用 `bsdf_microfacet_ggx_glass_setup` 初始化含折射的微表面模型

### precompute_ggx_gen_schlick_s()
- **签名**: `static float precompute_ggx_gen_schlick_s(const float rough, const float mu, const float eta, const float exponent, const float3 rand)`
- **功能**: 计算广义 Schlick Fresnel 近似下的 F0 到 F90 插值因子。利用双通道编码技巧（绿色通道编码 F0 对应值，红色通道编码 Fresnel 加权值），返回 `saturatef(eval.x / eval.y)` 作为归一化的 Fresnel 插值系数

### ior_parametrization()
- **签名**: `inline float ior_parametrization(const float z)`
- **功能**: 将 [0, 1] 范围的参数 z 映射到 [1, +inf) 的折射率（IOR）范围。采用 `ior_from_F0(z^4)` 的参数化方式，确保大部分精度分配在常见的 IOR 范围（1.0-2.0）

### cycles_precompute()
- **签名**: `static bool cycles_precompute(std::string name)`
- **功能**: 核心预计算调度函数，包含所有预计算项的定义和执行逻辑。支持以下预计算项：
  - **`ggx_E`** — GGX BRDF 总反照率（32x32 表，2^23 采样/点）
  - **`ggx_Eavg`** — GGX BRDF 平均反照率（32x1 表，2^26 采样/点），对余弦角进行加权积分
  - **`ggx_glass_E`** — GGX 玻璃 BSDF 总反照率，IOR>1（16x16x16 三维表，2^23 采样/点）
  - **`ggx_glass_Eavg`** — GGX 玻璃 BSDF 平均反照率，IOR>1（16x1x16 表，2^26 采样/点）
  - **`ggx_glass_inv_E`** — GGX 玻璃 BSDF 总反照率，IOR<1（使用 1/IOR）
  - **`ggx_glass_inv_Eavg`** — GGX 玻璃 BSDF 平均反照率，IOR<1
  - **`ggx_gen_schlick_ior_s`** — 广义 Schlick Fresnel（介电模式）插值因子（16x16x16 表）
  - **`ggx_gen_schlick_s`** — 广义 Schlick Fresnel 插值因子（16x16x16 表），指数参数从 [0,1] 重映射到 [0, +inf)

  使用 TBB `parallel_for` 对每个表格切片进行并行蒙特卡洛积分，采用 Sobol-Burley 低差异序列（`sobol_burley_sample_4D`）生成伪随机数以加速收敛。结果以 C++ 静态数组格式输出到标准输出

### main()
- **签名**: `int main(const int argc, const char **argv)`
- **功能**: 程序入口，接收一个命令行参数作为预计算项名称，调用 `cycles_precompute()` 执行对应的预计算任务

## 依赖关系

### 内部头文件
- `util/string.h` — 字符串工具
- `util/array.h` — 动态数组容器
- `util/hash.h` — 哈希函数（`hash_uint2`，用于生成每像素的随机种子）
- `util/tbb.h` — TBB 并行计算封装（`parallel_for`）
- `kernel/closure/bsdf_microfacet.h` — GGX 微表面 BSDF 实现（核心采样和评估函数）
- `kernel/sample/sobol_burley.h` — Sobol-Burley 低差异采样序列

### 外部库
- `<map>` — C++ 标准库
- `<iostream>` — 标准输出

### 被引用
- 该文件编译为独立可执行程序 `cycles_precompute`，不被其他源文件引用。其输出结果被手动复制到内核（Kernel）源码中的查找表定义

## 实现细节 / 关键算法

1. **蒙特卡洛积分**: 每个查找表元素通过对大量随机采样求平均来近似积分。对于反照率计算，积分量为 `eval/pdf`（重要性采样的标准估计量）。对于平均反照率（Eavg），额外乘以 `2*mu` 权重实现对入射余弦角的半球积分。

2. **Sobol-Burley 准随机序列**: 使用 `sobol_burley_sample_4D` 生成低差异序列，相比纯随机采样能显著降低方差、加速收敛。每个像素使用 `hash_uint2(x, y)` 生成独立种子以避免相关性。

3. **IOR 参数化**: 通过 `ior_from_F0(z^4)` 将均匀分布的 z 值映射到非均匀分布的 IOR 值，使得查找表在物理上更常见的低 IOR 区域（1.0-2.0，对应玻璃、水等常见材质）具有更高的精度。

4. **并行计算**: 使用 TBB 的 `parallel_for` 对表格的每个切片（固定 z 值）内的所有 (x, y) 像素并行计算，充分利用多核 CPU 资源。三维表格的 z 轴循环在外层串行执行。

5. **输出格式**: 结果直接以 C++ 源码格式（`static const float table_xxx[] = { ... };`）输出到标准输出，便于复制粘贴到内核源文件中。

## 关联文件

- `src/kernel/closure/bsdf_microfacet.h` — GGX 微表面 BSDF 的核心实现，提供 `bsdf_microfacet_ggx_setup`、`bsdf_microfacet_ggx_glass_setup` 和 `bsdf_microfacet_ggx_sample` 函数
- `src/kernel/sample/sobol_burley.h` — 准随机采样序列生成
- `src/app/CMakeLists.txt` — 通过 `WITH_CYCLES_PRECOMPUTE` 条件编译此目标
