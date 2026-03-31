# texture_3d.h - 三维纹理插值与体积数据采样

## 概述
本文件实现了三维纹理（主要是 NanoVDB 体积数据）的内核端采样和插值功能，是 Cycles 体积渲染管线的核心组件。文件支持最近邻、三线性和三次（tricubic）三种插值模式，并实现了基于随机采样的单点近似（stochastic one-tap sampling）优化。所有 NanoVDB 相关代码通过 `WITH_NANOVDB` 宏条件编译，在非 Metal/oneAPI 平台上启用。

## 类与结构体
本文件未定义类或结构体。

## 枚举与常量
本文件未定义枚举或常量，但使用以下来自外部的枚举值：
- **`InterpolationType`**: `INTERPOLATION_CLOSEST`、`INTERPOLATION_LINEAR`、`INTERPOLATION_CUBIC`、`INTERPOLATION_NONE`
- **`ImageDataType`**: `IMAGE_DATA_TYPE_NANOVDB_FLOAT`、`IMAGE_DATA_TYPE_NANOVDB_FLOAT3`、`IMAGE_DATA_TYPE_NANOVDB_FLOAT4`、`IMAGE_DATA_TYPE_NANOVDB_FPN`、`IMAGE_DATA_TYPE_NANOVDB_FP16`、`IMAGE_DATA_TYPE_NANOVDB_EMPTY`

## 核心函数

### fill_cubic_weights()
- **签名**: `ccl_device_inline void fill_cubic_weights(float3 w[4], float3 t)`
- **功能**: 计算三次 B 样条插值的四个权重值。输入 `t` 为小数部分坐标，输出 `w[0..3]` 为对应的插值权重。

### interp_tricubic_stochastic()
- **签名**: `ccl_device_inline float3 interp_tricubic_stochastic(const float3 P, ccl_private float3 &rand)`
- **功能**: 使用随机单点采样近似三次插值。基于蓄水池采样（reservoir sampling）从 4 个候选位置中随机选择一个采样点，权重为三次 B 样条权重。来源于论文 "Stochastic Texture Filtering"（2023）。

### interp_trilinear_stochastic()
- **签名**: `ccl_device_inline float3 interp_trilinear_stochastic(const float3 P, const float3 rand)`
- **功能**: 使用随机单点采样近似三线性插值。根据随机数与插值因子的比较，在两个相邻网格点间随机选择一个。

### interp_stochastic()
- **签名**: `ccl_device_inline float3 interp_stochastic(const float3 P, ccl_private InterpolationType &interpolation, ccl_private float3 &rand)`
- **功能**: 随机插值的统一调度函数。根据插值类型调用对应的随机采样函数，并将插值类型修改为 `INTERPOLATION_CLOSEST`（因为随机采样后只需最近邻查询即可）。

### kernel_tex_image_interp_trilinear_nanovdb()
- **签名**: `template<typename OutT, typename Acc> ccl_device OutT kernel_tex_image_interp_trilinear_nanovdb(ccl_private Acc &acc, const float3 P)`
- **功能**: 使用 NanoVDB 访问器进行三线性插值。对 2x2x2 = 8 个相邻体素进行加权平均。

### kernel_tex_image_interp_tricubic_nanovdb()
- **签名**: `template<typename OutT, typename Acc> ccl_device OutT kernel_tex_image_interp_tricubic_nanovdb(ccl_private Acc &acc, const float3 P)`
- **功能**: 使用 NanoVDB 访问器进行三次插值。对 4x4x4 = 64 个相邻体素进行加权求和，权重由 `fill_cubic_weights()` 计算。

### kernel_tex_image_interp_nanovdb()
- **签名**: `template<typename OutT, typename T> OutT kernel_tex_image_interp_nanovdb(const ccl_global TextureInfo &info, float3 P, const InterpolationType interp)`
- **功能**: NanoVDB 纹理插值的模板入口函数。根据插值类型选择最近邻（使用 `ReadAccessor`）、三线性或三次插值（后两者使用 `CachedReadAccessor` 以利用缓存加速多次查询）。标记为 `noinline` 以减少代码膨胀。

### kernel_tex_image_interp_3d()
- **签名**: `ccl_device float4 kernel_tex_image_interp_3d(KernelGlobals kg, ccl_private ShaderData *sd, const int id, float3 P, InterpolationType interp, const bool stochastic)`
- **功能**: 三维纹理采样的最终入口函数。处理完整的采样流程：坐标变换、随机采样、根据数据类型（float/float3/float4/FpN/Fp16/empty）分派到对应的 NanoVDB 模板实例。返回 `float4` 类型的 RGBA 颜色值。

## 依赖关系
- **内部头文件**:
  - `kernel/globals.h` — 提供 `KernelGlobals` 及全局数据访问
  - `kernel/sample/lcg.h` — 线性同余随机数生成器，用于随机采样
  - `util/texture.h` — `TextureInfo` 结构体和纹理相关类型
  - `kernel/util/nanovdb.h` — NanoVDB 数据结构和访问器（条件引入，排除 Metal 和 oneAPI 平台）
- **被引用**:
  - `kernel/osl/services_gpu.h` — GPU 端 OSL 着色器服务中的体积纹理查询
  - `kernel/osl/services.cpp` — CPU 端 OSL 着色器服务中的体积纹理查询
  - `kernel/geom/volume.h` — 体积几何处理中的密度和属性采样

## 实现细节 / 关键算法

### 随机纹理过滤
文件实现了基于论文 "Stochastic Texture Filtering"（arXiv:2305.05810）的随机采样技术。核心思想是将多点插值转化为单点随机采样：
- **三次随机采样**: 使用蓄水池采样策略，依次遍历 4 个候选位置，以三次 B 样条权重为概率接受/拒绝。随机数在每次迭代中被重新归一化以保持分层特性。
- **线性随机采样**: 以插值因子为概率在两个相邻点间选择。
- 随机采样后将插值类型改为最近邻，因此实际的 NanoVDB 查询只需访问单个体素。

### 三次 B 样条插值
权重公式为标准的三次均匀 B 样条核，四个权重分别为：
- `w[0] = ((-1/6)t + 1/2)t - 1/2)t + 1/6`
- `w[1] = ((1/2)t - 1)t)t + 2/3`
- `w[2] = ((-1/2)t + 1/2)t + 1/2)t + 1/6`
- `w[3] = (1/6)t^3`

### 访问器选择策略
- **最近邻**: 使用无缓存的 `ReadAccessor`，因为只需单次查询
- **线性/三次**: 使用带缓存的 `CachedReadAccessor`，因为需要对空间邻域内的多个体素进行查询，缓存可大幅提升性能

### GPU 符号隔离
在非 GPU 平台上，模板函数被包裹在匿名命名空间 `namespace {}` 中，以防止不同指令集（如 SSE/AVX）编译的内核之间发生符号冲突。

### 数据类型分派
`kernel_tex_image_interp_3d()` 根据 `ImageDataType` 枚举值分派到不同的模板实例化：
- `NANOVDB_FLOAT` — 单通道浮点体积（如密度、温度）
- `NANOVDB_FLOAT3` — 三通道体积（如速度场）
- `NANOVDB_FLOAT4` — 四通道体积
- `NANOVDB_FPN` — N 位量化压缩体积
- `NANOVDB_FP16` — 16 位量化压缩体积
- `NANOVDB_EMPTY` — 空网格，直接返回零值

## 关联文件
- `src/kernel/util/nanovdb.h` — NanoVDB 数据结构定义
- `src/kernel/geom/volume.h` — 体积几何体采样，调用本文件的函数
- `src/kernel/osl/services.cpp` — OSL 体积纹理接口
- `src/util/texture.h` — `TextureInfo` 定义和纹理枚举类型
- `src/kernel/sample/lcg.h` — 随机数生成器
