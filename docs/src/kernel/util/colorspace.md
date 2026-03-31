# colorspace.h - 内核色彩空间转换工具

## 概述
本文件提供在 Cycles 渲染内核中进行各种色彩空间转换的设备端函数。它负责 XYZ、Rec.709、线性 RGB 以及 Spectrum（光谱）之间的相互转换，是着色器计算、胶片（Film）输出和光照积分过程中不可或缺的基础工具。转换矩阵系数存储在 `kernel_data.film` 中，在场景初始化阶段由主机端写入，从而支持不同的输出色彩空间配置。

## 类与结构体
本文件未定义类或结构体。

## 枚举与常量
本文件未定义枚举或常量。

## 核心函数

### xyz_to_rgb()
- **签名**: `ccl_device float3 xyz_to_rgb(KernelGlobals kg, const float3 xyz)`
- **功能**: 将 CIE XYZ 色彩空间的颜色值转换为当前渲染配置的 RGB 色彩空间。通过与 `kernel_data.film` 中存储的 `xyz_to_r/g/b` 三组转换向量分别做点积来完成矩阵乘法。

### xyz_to_rgb_clamped()
- **签名**: `ccl_device float3 xyz_to_rgb_clamped(KernelGlobals kg, const float3 xyz)`
- **功能**: 与 `xyz_to_rgb()` 相同，但将结果钳制到非负值（最小为 0），防止因色域映射产生负数通道。

### rec709_to_rgb()
- **签名**: `ccl_device float3 rec709_to_rgb(KernelGlobals kg, const float3 rec709)`
- **功能**: 将 Rec.709 色彩空间的颜色转换为当前渲染 RGB 色彩空间。若当前渲染色彩空间本身就是 Rec.709（`kernel_data.film.is_rec709` 为真），则直接返回原值以跳过不必要的矩阵运算。

### linear_rgb_to_gray()
- **签名**: `ccl_device float linear_rgb_to_gray(KernelGlobals kg, const float3 c)`
- **功能**: 将线性 RGB 颜色转换为灰度值（亮度）。使用 `kernel_data.film.rgb_to_y` 中存储的亮度权重系数与输入颜色做点积。

### rgb_to_spectrum()
- **签名**: `ccl_device_inline Spectrum rgb_to_spectrum(const float3 rgb)`
- **功能**: 将 RGB 值转换为光谱类型（Spectrum）。在当前 RGB 光谱模式下为恒等变换，直接返回输入值。此函数为光谱渲染提供了抽象接口。

### spectrum_to_rgb()
- **签名**: `ccl_device_inline float3 spectrum_to_rgb(Spectrum s)`
- **功能**: 将光谱类型转换回 RGB 值。在当前 RGB 光谱模式下为恒等变换。

### spectrum_to_gray()
- **签名**: `ccl_device float spectrum_to_gray(KernelGlobals kg, Spectrum c)`
- **功能**: 将光谱值先转为 RGB 再转为灰度值。组合调用 `spectrum_to_rgb()` 和 `linear_rgb_to_gray()` 实现。

## 依赖关系
- **内部头文件**:
  - `kernel/globals.h` — 提供 `KernelGlobals` 类型及 `kernel_data` 全局数据访问
  - `util/types_spectrum.h` — 提供 `Spectrum` 类型定义
- **被引用**: 本文件被内核中大量着色器和积分器模块引用，包括：
  - `kernel/svm/wavelength.h`、`kernel/svm/sky.h`、`kernel/svm/blackbody.h`、`kernel/svm/closure.h`、`kernel/svm/convert.h` — 着色器虚拟机（SVM）节点
  - `kernel/integrator/guiding.h` — 路径引导
  - `kernel/film/write.h` — 胶片写入
  - `kernel/geom/attribute.h` — 几何属性
  - `kernel/closure/bsdf_util.h`、`kernel/closure/bsdf_diffuse_ramp.h`、`kernel/closure/bsdf_phong_ramp.h`、`kernel/closure/bsdf_principled_hair_chiang.h` — BSDF 闭包
  - `kernel/bake/bake.h` — 烘焙

## 实现细节 / 关键算法
- 色彩空间转换采用 3x3 矩阵乘法实现，矩阵的每行以 `float3` 形式存储在 `kernel_data.film` 结构中，通过三次点积运算完成。
- `rec709_to_rgb()` 中包含快速路径优化：若渲染色彩空间即为 Rec.709，则跳过矩阵运算直接返回。
- `rgb_to_spectrum()` 和 `spectrum_to_rgb()` 当前为恒等函数，这是因为 Cycles 当前使用 RGB 作为光谱表示。这些函数的存在为未来引入全光谱渲染（如使用更复杂的光谱上采样方法）预留了接口。

## 关联文件
- `src/kernel/globals.h` — `KernelGlobals` 及 `kernel_data` 定义
- `src/util/types_spectrum.h` — `Spectrum` 类型
- `src/kernel/film/write.h` — 胶片写入中使用色彩空间转换
- `src/scene/colorspace.cpp` — 主机端色彩空间管理，负责生成写入 `kernel_data.film` 的转换矩阵
