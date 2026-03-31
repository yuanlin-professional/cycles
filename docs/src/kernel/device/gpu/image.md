# image.h - GPU 纹理图像插值采样

## 概述

本文件实现了 Cycles 渲染器在 GPU 设备上的纹理图像采样功能。核心提供了双三次（bicubic）插值算法的高效 GPU 实现，利用 4 次双线性查找模拟一次完整的双三次采样，大幅减少了纹理读取次数。同时提供了统一的纹理采样入口函数，根据纹理数据类型和插值模式自动选择合适的采样路径。

## 核心函数

### frac()
- **签名**: `ccl_device_inline float frac(const float x, ccl_private int *ix)`
- **功能**: 计算浮点数的小数部分并输出整数部分。对负数进行特殊处理，确保小数部分始终为非负值。用于将连续纹理坐标分解为整数像素索引和子像素偏移量。

### cubic_w0() / cubic_w1() / cubic_w2() / cubic_w3()
- **签名**: `ccl_device float cubic_w0(const float a)` （w1/w2/w3 签名类似）
- **功能**: 三次 B 样条的四个基函数。输入参数 `a` 为子像素偏移（0 到 1 之间），返回对应的权重值。这四个函数共同构成了双三次插值的权重基础。

### cubic_g0() / cubic_g1()
- **签名**: `ccl_device float cubic_g0(const float a)`
- **功能**: 振幅函数。`g0 = w0 + w1`，`g1 = w2 + w3`。将四个权重合并为两组，用于将 4x4 的双三次查找优化为 2x2 的双线性查找。

### cubic_h0() / cubic_h1()
- **签名**: `ccl_device float cubic_h0(const float a)`
- **功能**: 偏移函数。根据权重比值计算调整后的采样坐标偏移量，使得硬件双线性插值的结果等价于双三次插值中相邻两个采样点的加权平均。

### kernel_tex_image_interp_bicubic<T>()
- **签名**: `template<typename T> ccl_device_noinline T kernel_tex_image_interp_bicubic(const ccl_global TextureInfo &info, float x, float y)`
- **功能**: 快速双三次纹理插值的核心实现。利用上述辅助函数，仅通过 4 次硬件双线性纹理查找（而非朴素实现的 16 次点采样）完成一次双三次采样。模板参数 `T` 支持 `float` 和 `float4` 类型。

### kernel_tex_image_interp()
- **签名**: `ccl_device float4 kernel_tex_image_interp(KernelGlobals kg, const int id, const float x, float y)`
- **功能**: GPU 纹理采样的统一入口。根据纹理数据类型（float4/byte4/ushort4/half4 或 float/byte/half）和插值模式（三次/智能/线性）分发到对应的采样路径。单通道纹理的采样结果会扩展为 `float4(f, f, f, 1.0f)` 格式返回。

## 依赖关系

- **内部头文件**:
  - `kernel/globals.h` - 内核全局数据和 `TextureInfo` 结构体定义
- **被引用**:
  - `kernel/device/optix/kernel.cu` - OptiX 光线追踪设备内核
  - `kernel/osl/services_optix.cu` - OptiX 的 OSL 着色服务

## 实现细节

### 快速双三次插值优化

传统的双三次插值需要对 4x4 邻域共 16 个像素进行采样，这在 GPU 上代价高昂。本文件采用了一种经典的优化技巧（源自 CUDA Samples）：

1. 将 4 个三次 B 样条基函数 (w0-w3) 两两合并为振幅函数 (g0, g1)
2. 计算使硬件线性插值等效的偏移坐标 (h0, h1)
3. 仅需 4 次 GPU 硬件双线性纹理查找即可得到精确的双三次结果
4. 最终结果为 `g0(fy) * [g0(fx) * tex(x0,y0) + g1(fx) * tex(x1,y0)] + g1(fy) * [g0(fx) * tex(x0,y1) + g1(fx) * tex(x1,y1)]`

注意坐标中有 `+0.5f` 偏移，用于补偿 CUDA 线性过滤的坐标约定。

### 数据类型分发

`kernel_tex_image_interp()` 根据 `TextureInfo::data_type` 区分两大类纹理：
- **多通道纹理**（float4/byte4/ushort4/half4）：直接返回 `float4` 结果
- **单通道纹理**（float/byte/half）：采样得到标量后扩展为灰度 `float4`

两类纹理都支持三次插值（`INTERPOLATION_CUBIC`）和智能插值（`INTERPOLATION_SMART`），否则回退到硬件双线性插值。

## 关联文件

| 文件 | 关系 |
|------|------|
| `kernel/globals.h` | 提供 `TextureInfo` 结构体和全局数据访问宏 |
| `kernel/device/optix/kernel.cu` | OptiX 后端包含本文件实现纹理采样 |
| `kernel/osl/services_optix.cu` | OptiX OSL 着色服务使用本文件 |
| `kernel/device/gpu/kernel.h` | GPU 通用内核（不直接包含本文件，但属于同层级模块） |
