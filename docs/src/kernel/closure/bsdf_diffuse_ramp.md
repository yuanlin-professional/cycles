# bsdf_diffuse_ramp.h - 渐变漫反射 BSDF 模型

## 概述

`bsdf_diffuse_ramp.h` 实现了基于颜色渐变（Ramp）的漫反射 BSDF 模型。与标准 Lambertian 漫反射使用单一颜色不同，该模型根据出射方向与法线的夹角余弦值在一个 8 色渐变条中插值获取颜色。这为艺术家提供了对漫反射响应曲线的精细控制，适合非真实感渲染或特殊材质效果。此闭包仅在 OSL 模式下可用。

## 类与结构体

### DiffuseRampBsdf
```c
struct DiffuseRampBsdf {
    SHADER_CLOSURE_BASE;
    ccl_private float3 *colors;  // 指向 8 个 RGB 颜色值的数组
};
```

`colors` 指针指向一个长度为 8 的 `float3` 数组，通过 `closure_alloc_extra` 分配额外内存存储。渐变条从 `colors[0]`（N.wo = 0，掠射角）到 `colors[7]`（N.wo = 1，法线方向）进行线性插值。

## 核心函数

### bsdf_diffuse_ramp_get_color()
- **签名**: `ccl_device float3 bsdf_diffuse_ramp_get_color(const float3 colors[8], float pos)`
- **功能**: 根据参数 `pos`（范围 [0, 1]）在 8 色渐变条中进行线性插值。将 `pos` 映射到 [0, 7] 的浮点索引，然后在相邻两个颜色之间线性混合。边界处返回端点颜色。

### bsdf_diffuse_ramp_setup()
- **签名**: `ccl_device int bsdf_diffuse_ramp_setup(DiffuseRampBsdf *bsdf)`
- **功能**: 初始化渐变漫反射闭包，设置类型为 `CLOSURE_BSDF_DIFFUSE_RAMP_ID`。返回 `SD_BSDF | SD_BSDF_HAS_EVAL`。

### bsdf_diffuse_ramp_blur()
- **签名**: `ccl_device void bsdf_diffuse_ramp_blur(ccl_private ShaderClosure *sc, const float roughness)`
- **功能**: 空操作（no-op）。渐变漫反射不支持模糊操作。

### bsdf_diffuse_ramp_eval()
- **签名**: `ccl_device Spectrum bsdf_diffuse_ramp_eval(const ccl_private ShaderClosure *sc, const float3 wi, const float3 wo, ccl_private float *pdf)`
- **功能**: 计算渐变漫反射 BSDF 值。PDF 使用标准余弦分布 `cos(N, wo) / pi`。BSDF 颜色通过 `bsdf_diffuse_ramp_get_color(colors, cosNO)` 查询渐变条获得，然后乘以 `1/pi` 归一化。

### bsdf_diffuse_ramp_sample()
- **签名**: `ccl_device int bsdf_diffuse_ramp_sample(const ccl_private ShaderClosure *sc, const float3 Ng, const float3 wi, const float2 rand, ccl_private Spectrum *eval, ccl_private float3 *wo, ccl_private float *pdf)`
- **功能**: 使用余弦加权半球采样生成出射方向。采样颜色通过将 PDF 乘以 pi 还原为余弦值后查询渐变条。返回 `LABEL_REFLECT | LABEL_DIFFUSE`。

## 依赖关系

- **内部头文件**:
  - `kernel/types.h` — 核心类型定义
  - `kernel/sample/mapping.h` — 提供 `sample_cos_hemisphere` 采样函数
  - `kernel/util/colorspace.h` — 提供 `rgb_to_spectrum` 颜色空间转换

- **被引用**:
  - `kernel/closure/bsdf.h` — 通过统一调度接口调用（在 `__OSL__` 宏条件下）

## 实现细节 / 关键算法

### 渐变条查表

颜色查询使用简单的线性插值方案：
1. 将 `pos` 映射到索引范围 `pos * 7`
2. 取整数部分作为基础索引 `ipos`
3. 小数部分作为插值权重 `offset`
4. 输出 `colors[ipos] * (1 - offset) + colors[ipos + 1] * offset`

### 条件编译

整个实现被 `#ifdef __OSL__` 包裹，仅在 OSL 模式下可用。这是因为此闭包由 OSL 着色器系统通过 `closure_alloc_extra` 机制分配颜色数据。

## 关联文件

- `kernel/closure/bsdf.h` — 统一调度中心
- `kernel/closure/bsdf_phong_ramp.h` — 姊妹实现，使用相同渐变条机制但采用 Phong 高光模型
- `kernel/osl/closures_setup.h` — OSL 闭包创建
