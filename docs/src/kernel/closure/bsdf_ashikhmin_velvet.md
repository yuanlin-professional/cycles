# bsdf_ashikhmin_velvet.h - Ashikhmin 天鹅绒 BSDF 模型

## 概述

`bsdf_ashikhmin_velvet.h` 实现了一种天鹅绒（Velvet）材质的双向散射分布函数（BSDF），用于模拟天鹅绒、绒布等具有逆反射特征的织物材质。该模型基于 Open Shading Language 的适配代码，使用高斯分布的法线分布函数（NDF）和几何遮蔽项来产生边缘增亮效果，属于漫反射类别。

## 类与结构体

### VelvetBsdf
```c
struct VelvetBsdf {
    SHADER_CLOSURE_BASE;
    float sigma;        // 表面粗糙度参数（最小值 0.01）
    float invsigma2;    // 预计算的 1/(sigma^2)，用于加速求值
};
```

继承自 `SHADER_CLOSURE_BASE` 宏定义的基类字段（包括 `type`、`weight`、`sample_weight`、`N` 等）。通过 `static_assert` 确保其大小不超过 `ShaderClosure`。

## 核心函数

### bsdf_ashikhmin_velvet_setup()
- **签名**: `ccl_device int bsdf_ashikhmin_velvet_setup(ccl_private VelvetBsdf *bsdf)`
- **功能**: 初始化天鹅绒闭包。将 `sigma` 限制为至少 0.01，预计算 `invsigma2 = 1/(sigma^2)`，设置闭包类型为 `CLOSURE_BSDF_ASHIKHMIN_VELVET_ID`。返回 `SD_BSDF | SD_BSDF_HAS_EVAL` 标志。

### bsdf_ashikhmin_velvet_eval()
- **签名**: `ccl_device Spectrum bsdf_ashikhmin_velvet_eval(const ccl_private ShaderClosure *sc, const float3 wi, const float3 wo, ccl_private float *pdf)`
- **功能**: 计算给定入射/出射方向对的 BSDF 值和 PDF。核心计算包含：高斯法线分布函数 `D = exp(-cot^2 / sigma^2) / (sigma^2 * pi * sin^4(N.H))`，以及 Cook-Torrance 式几何遮蔽项 `G = min(1, 2*|N.H/H.I * N.I|, 2*|N.H/H.I * N.O|)`。PDF 使用均匀半球分布 `0.5/pi`。

### bsdf_ashikhmin_velvet_sample()
- **签名**: `ccl_device int bsdf_ashikhmin_velvet_sample(const ccl_private ShaderClosure *sc, const float3 Ng, const float3 wi, const float2 rand, ccl_private Spectrum *eval, ccl_private float3 *wo, ccl_private float *pdf)`
- **功能**: 使用均匀半球采样生成出射方向。采样后使用与 `eval` 相同的公式计算 BSDF 值。返回标签 `LABEL_REFLECT | LABEL_DIFFUSE`。

## 依赖关系

- **内部头文件**:
  - `kernel/types.h` — 核心类型定义
  - `kernel/sample/mapping.h` — 提供 `sample_uniform_hemisphere` 采样函数

- **被引用**:
  - `kernel/closure/bsdf.h` — 通过统一调度接口调用

## 实现细节 / 关键算法

### 天鹅绒 BRDF 公式

最终输出为 `0.25 * D * G / cos(N, wi)`，其中：

- **法线分布函数 D**: 使用余切角的高斯分布 `exp(-cot^2(N.H) / sigma^2) * (1/sigma^2) * (1/pi) / sin^4(N.H)`。`sigma` 参数控制高光的宽窄：较小的 sigma 产生更集中的边缘高光，较大的 sigma 产生更弥散的效果。
- **几何遮蔽项 G**: 使用 Cook-Torrance 的 V 型凹槽近似 `min(1, fac1, fac2)`。代码中标注了 TODO：未来可能从 D 解析推导 G。
- **采样策略**: 使用均匀半球采样（而非重要性采样），因此 PDF 恒定为 `1/(2*pi)`。这种简单的采样方式虽然不是最优的，但对天鹅绒类材质足够有效。

## 关联文件

- `kernel/closure/bsdf.h` — 统一调度中心
- `kernel/sample/mapping.h` — 半球采样工具
- `kernel/svm/closure.h` — SVM 中创建天鹅绒闭包
