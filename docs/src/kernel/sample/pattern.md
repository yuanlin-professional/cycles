# pattern.h - 采样模式调度与路径随机数生成

## 概述
本文件是 Cycles 采样子系统的核心调度层，负责为路径追踪内核提供高质量的伪随机数序列。它根据用户选择的采样模式（Tabulated Sobol、Sobol-Burley、蓝噪声等）将随机数生成请求分发到对应的底层实现。同时负责像素级随机种子初始化和自适应采样所需的样本分类。该文件是积分器与底层采样器之间的桥梁。

## 类与结构体
无。

## 枚举与常量
文件引用了以下采样模式枚举（定义在其他头文件中）：

| 枚举值 | 说明 |
|--------|------|
| `SAMPLING_PATTERN_TABULATED_SOBOL` | 预计算表格 Sobol 采样器 |
| `SAMPLING_PATTERN_SOBOL_BURLEY` | Sobol-Burley 扰乱采样器 |
| `SAMPLING_PATTERN_BLUE_NOISE_PURE` | 纯蓝噪声采样模式 |
| `SAMPLING_PATTERN_BLUE_NOISE_FIRST` | 首帧蓝噪声 + 后续混合模式 |

调试宏：
- `__DEBUG_CORRELATION__`（默认注释掉）：启用时所有采样函数退化为 `drand48()`，用于单线程 CPU 调试相关性问题。

## 核心函数

### blue_noise_indexing()
- **签名**: `ccl_device_forceinline uint3 blue_noise_indexing(KernelGlobals kg, uint pixel_index, const uint sample)`
- **功能**: 根据当前采样模式为蓝噪声相关模式计算索引三元组 `(sample_index, seed, index_mask)`。不同模式的索引策略如下：
  - `SOBOL_BURLEY`：每个像素独立序列，使用长度掩码优化
  - `BLUE_NOISE_PURE`：所有像素共享单一序列，每个像素在序列中偏移
  - `BLUE_NOISE_FIRST`：首个样本使用 1SPP 蓝噪声序列，后续样本使用独立的 N-1 SPP 序列，专为视口导航时的 1SPP 质量优化

### path_rng_1D()
- **签名**: `ccl_device_forceinline float path_rng_1D(KernelGlobals kg, const uint rng_pixel, const uint sample, const int dimension)`
- **功能**: 生成一维路径随机数。根据采样模式分发到 `tabulated_sobol_sample_1D` 或 `sobol_burley_sample_1D`。

### path_rng_2D()
- **签名**: `ccl_device_forceinline float2 path_rng_2D(KernelGlobals kg, const uint rng_pixel, const int sample, const int dimension)`
- **功能**: 生成二维路径随机数。维度内部具有分层性，适用于需要联合二维分层的场景（如圆盘采样、光源采样等）。

### path_rng_3D()
- **签名**: `ccl_device_forceinline float3 path_rng_3D(KernelGlobals kg, const uint rng_pixel, const int sample, const int dimension)`
- **功能**: 生成三维路径随机数。

### path_rng_4D()
- **签名**: `ccl_device_forceinline float4 path_rng_4D(KernelGlobals kg, const uint rng_pixel, const int sample, const int dimension)`
- **功能**: 生成四维路径随机数。

### path_rng_pixel_init()
- **签名**: `ccl_device_inline uint path_rng_pixel_init(KernelGlobals kg, const int sample, const int x, const int y)`
- **功能**: 初始化像素级随机数种子。不同采样模式使用不同策略：
  - **白噪声采样器**（Tabulated Sobol、Sobol-Burley）：使用 `hash_iqnt2d(x,y)` 与全局种子异或，每个像素获得独立随机哈希
  - **蓝噪声采样器**：使用分层洗牌的二维 Morton 曲线确定每个像素在序列中的偏移，基于 Owen 扰乱实现分层抖动

### sample_is_class_A()
- **签名**: `ccl_device_inline bool sample_is_class_A(const int pattern, const int sample)`
- **功能**: 将样本分为 A/B 两类，用于自适应采样中的方差估计。基于 Christensen 等人的渐进多抖动采样序列论文（第 10.2.1 节），通过位运算 `popcount(sample & 0xaaaaaaaa) & 1` 高效实现。该分类方式同样适用于 Owen 扰乱和洗牌 Sobol 序列。

## 依赖关系
- **内部头文件**:
  - `kernel/globals.h` - 内核全局数据结构
  - `kernel/sample/sobol_burley.h` - Sobol-Burley 采样实现
  - `kernel/sample/tabulated_sobol.h` - 预计算表格 Sobol 采样实现
  - `util/hash.h` - 哈希函数集
- **被引用**:
  - `src/kernel/integrator/init_from_bake.h` - 烘焙模式积分器初始化
  - `src/kernel/integrator/init_from_camera.h` - 相机模式积分器初始化
  - `src/kernel/integrator/path_state.h` - 路径状态管理
  - `src/kernel/film/light_passes.h` - 胶片光照通道处理

## 实现细节 / 关键算法

### Sobol 采样维度策略
文件注释中说明了 Sobol 序列的维度使用准则：
1. **低维优先原则**：较低维度的分层性更好，应优先分配给最重要的采样任务
2. **相邻维度原则**：相邻维度之间的联合分层性优于间隔较远的维度，应将需要联合采样的变量安排在相邻维度

### 蓝噪声像素索引
蓝噪声采样器使用分层洗牌的二维 Morton 曲线（Z 阶空间填充曲线）来确定像素的序列偏移。Morton 编码将二维像素坐标映射为一维索引，保持空间局部性。然后通过 base-4 Owen 扰乱对 Morton 索引进行洗牌，使相邻像素获得充分装饰的序列偏移。该方法基于 psychopath.io 的蓝噪声抖动采样论文。

### 自适应采样的样本分类
`sample_is_class_A` 使用 `popcount(sample & 0xAAAAAAAA)` 的奇偶性来分类。掩码 `0xAAAAAAAA` 选取偶数位，`popcount` 计数被置位的数量。这种分类方式保证了 A 类和 B 类样本各自保持良好的分层性，从而使方差估计更加可靠。该技术最初用于渐进多抖动序列，经验证同样适用于 Owen 扰乱 Sobol。

### BLUE_NOISE_FIRST 混合策略
视口导航时通常只渲染 1 SPP。纯蓝噪声序列的优势在于多像素间的样本分布呈蓝噪声特性，但仅取序列首个样本无法获得该优势。因此 `BLUE_NOISE_FIRST` 为第一个样本使用专门的 1SPP 蓝噪声序列（种子 `0x0cd0519f`），后续样本使用标准蓝噪声序列，在 1SPP 和全 SPP 质量之间取得平衡。

## 关联文件
- `src/kernel/sample/sobol_burley.h` - Sobol-Burley 采样器实现
- `src/kernel/sample/tabulated_sobol.h` - 预计算表格 Sobol 采样器实现
- `src/kernel/sample/util.h` - Owen 扰乱和 Morton 编码工具
- `src/kernel/integrator/init_from_camera.h` - 调用 `path_rng_pixel_init` 初始化像素种子
- `src/kernel/integrator/path_state.h` - 调用 `path_rng_*D` 生成路径随机数
- `src/kernel/film/light_passes.h` - 调用 `sample_is_class_A` 进行自适应采样方差估计
