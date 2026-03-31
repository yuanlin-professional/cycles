# kernel/closure - 闭包/双向散射分布函数 (BSDF) 实现

## 概述

`src/kernel/closure/` 模块实现了 Cycles 路径追踪渲染器中所有**闭包（Closure）**的内核端代码。闭包是渲染方程中描述光与表面/体积交互的数学模型，包括双向散射分布函数（BSDF）、次表面散射（BSSRDF）、发射（Emission）和体积散射相位函数（Phase Function）。

每个闭包类型都实现了标准的三个操作接口：
- **`setup()`**：初始化闭包参数，返回着色器标志
- **`eval()`**：给定入射和出射方向，计算闭包的值与概率密度函数（PDF）
- **`sample()`**：给定入射方向和随机数，重要性采样一个出射方向

这些闭包由着色器评估（SVM/OSL）创建，并在积分器的路径追踪循环中通过统一分派函数 `bsdf_sample()` 和 `bsdf_eval()` 调用。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `alloc.h` | 基础设施 | 闭包内存分配：`closure_alloc()`、`closure_alloc_extra()`、`bsdf_alloc()` |
| `bsdf.h` | 统一分派 | BSDF 统一入口：`bsdf_sample()`、`bsdf_eval()`、`bsdf_label()`、`bsdf_blur()`、`bsdf_albedo()`；包含 bump 阴影项和阴影终止器偏移 |
| `bsdf_util.h` | 工具 | 菲涅尔计算（`fresnel_dielectric_polarized`）、薄膜干涉（`FresnelThinFilm`）、复数折射率类型 |
| **漫反射类** | | |
| `bsdf_diffuse.h` | 漫反射 | `DiffuseBsdf`：Lambertian 漫反射 + `bsdf_translucent_*`：透射漫反射 |
| `bsdf_oren_nayar.h` | 漫反射 | `OrenNayarBsdf`：改进版 Oren-Nayar 漫反射模型（Fujii 2012），含多重散射能量守恒项 |
| `bsdf_burley.h` | 漫反射 | `BurleyBsdf`：Burley/Disney 漫反射（仅 OSL）|
| `bsdf_diffuse_ramp.h` | 漫反射 | `DiffuseRampBsdf`：带色彩渐变的漫反射（仅 OSL）|
| **光泽反射类** | | |
| `bsdf_microfacet.h` | 微表面 | `MicrofacetBsdf`：GGX 和 Beckmann 微表面模型，支持反射/折射/玻璃模式，支持各向异性、薄膜干涉、多种菲涅尔模型（介电体、导体、广义 Schlick、F82 Tint）、能量补偿 |
| `bsdf_ashikhmin_shirley.h` | 光泽 | Ashikhmin-Shirley 各向异性光泽反射 |
| `bsdf_ashikhmin_velvet.h` | 光泽 | Ashikhmin 天鹅绒反射模型 |
| `bsdf_phong_ramp.h` | 光泽 | `PhongRampBsdf`：带色彩渐变的 Phong 反射（仅 OSL）|
| `bsdf_sheen.h` | 光泽 | `SheenBsdf`：实用多重散射光泽模型（Zeltner 2022），使用线性变换余弦（LTC）|
| `bsdf_toon.h` | 卡通 | `ToonBsdf`：卡通漫反射 + 卡通光泽着色 |
| **透射类** | | |
| `bsdf_transparent.h` | 透射 | 完全透明 BSDF（不改变方向的透射）|
| `bsdf_ray_portal.h` | 传送 | `RayPortalClosure`：射线传送门闭包，用于空间传送效果 |
| **毛发类** | | |
| `bsdf_hair.h` | 毛发 | `HairBsdf`：基础毛发反射/透射模型 |
| `bsdf_principled_hair_chiang.h` | 毛发 | `ChiangHairBSDF`：Chiang 2015 Principled Hair 模型（SIGGRAPH 2016）|
| `bsdf_principled_hair_huang.h` | 毛发 | `HuangHairBSDF`：Huang 2022 微表面毛发模型 |
| **次表面散射** | | |
| `bssrdf.h` | SSS | `Bssrdf`：次表面散射闭包，支持 Burley 扩散近似和随机游走（Random Walk / Random Walk Skin）|
| **发射** | | |
| `emissive.h` | 发射 | `emission_setup()`、`background_setup()`：表面发射和背景发射闭包 |
| **体积散射** | | |
| `volume.h` | 体积入口 | 体积闭包统一入口：`volume_phase_eval()`、`volume_phase_sample()`、`volume_extinction_setup()`；体积采样通道选择 |
| `volume_henyey_greenstein.h` | 体积相函数 | `HenyeyGreensteinVolume`：Henyey-Greenstein 相位函数（参数 g 控制前/后向散射）|
| `volume_rayleigh.h` | 体积相函数 | `RayleighVolume`：Rayleigh 散射相位函数（大气散射）|
| `volume_draine.h` | 体积相函数 | `DraineVolume`：Draine 相位函数（星际尘埃散射，参数 g 和 alpha）|
| `volume_fournier_forand.h` | 体积相函数 | `FournierForandVolume`：Fournier-Forand 相位函数（水下散射，参数 B 和 IOR）|
| `volume_util.h` | 体积工具 | 相位函数采样工具：`phase_sample_direction()`、`phase_henyey_greenstein()` 基础 HG 函数 |

## 核心类与数据结构

### 闭包基础结构

所有表面闭包继承自 `SHADER_CLOSURE_BASE` 宏（在 `ShaderClosure` 中定义），包含：
- **`type`** (`ClosureType`)：闭包类型枚举
- **`weight`** (`Spectrum`)：闭包权重/颜色
- **`sample_weight`** (`float`)：采样权重
- **`N`** (`float3`)：着色法线

体积闭包继承自 `SHADER_CLOSURE_VOLUME_BASE`，存储在独立的 `ShaderVolumeClosure` 结构中。

### 闭包分类

#### 漫反射闭包 (Diffuse)

| 闭包 ID | 数据结构 | 模型 |
|---------|---------|------|
| `CLOSURE_BSDF_DIFFUSE_ID` | `DiffuseBsdf` | Lambertian 漫反射 (cos/pi) |
| `CLOSURE_BSDF_OREN_NAYAR_ID` | `OrenNayarBsdf` | 改进版 Oren-Nayar（含多重散射）|
| `CLOSURE_BSDF_TRANSLUCENT_ID` | `DiffuseBsdf` | 透射漫反射 |
| `CLOSURE_BSDF_BURLEY_ID` | `BurleyBsdf` | Burley/Disney 漫反射（仅 OSL）|
| `CLOSURE_BSDF_DIFFUSE_RAMP_ID` | `DiffuseRampBsdf` | 渐变漫反射（仅 OSL）|

#### 光泽/微表面闭包 (Glossy/Microfacet)

| 闭包 ID | 数据结构 | 模型 |
|---------|---------|------|
| `CLOSURE_BSDF_MICROFACET_GGX_ID` | `MicrofacetBsdf` | GGX 反射 |
| `CLOSURE_BSDF_MICROFACET_GGX_REFRACTION_ID` | `MicrofacetBsdf` | GGX 折射 |
| `CLOSURE_BSDF_MICROFACET_GGX_GLASS_ID` | `MicrofacetBsdf` | GGX 玻璃（反射+折射）|
| `CLOSURE_BSDF_MICROFACET_BECKMANN_ID` | `MicrofacetBsdf` | Beckmann 反射 |
| `CLOSURE_BSDF_MICROFACET_BECKMANN_REFRACTION_ID` | `MicrofacetBsdf` | Beckmann 折射 |
| `CLOSURE_BSDF_MICROFACET_BECKMANN_GLASS_ID` | `MicrofacetBsdf` | Beckmann 玻璃 |
| `CLOSURE_BSDF_ASHIKHMIN_SHIRLEY_ID` | `MicrofacetBsdf` | Ashikhmin-Shirley 各向异性 |
| `CLOSURE_BSDF_ASHIKHMIN_VELVET_ID` | — | Ashikhmin 天鹅绒 |
| `CLOSURE_BSDF_SHEEN_ID` | `SheenBsdf` | 多重散射光泽（LTC 方法）|
| `CLOSURE_BSDF_PHONG_RAMP_ID` | `PhongRampBsdf` | Phong 渐变（仅 OSL）|
| `CLOSURE_BSDF_DIFFUSE_TOON_ID` | `ToonBsdf` | 卡通漫反射 |
| `CLOSURE_BSDF_GLOSSY_TOON_ID` | `ToonBsdf` | 卡通光泽 |

#### 透射/传送闭包 (Transmission/Portal)

| 闭包 ID | 数据结构 | 模型 |
|---------|---------|------|
| `CLOSURE_BSDF_TRANSPARENT_ID` | `ShaderClosure` | 完全透明透射 |
| `CLOSURE_BSDF_RAY_PORTAL_ID` | `RayPortalClosure` | 射线传送门 |

#### 毛发闭包 (Hair)

| 闭包 ID | 数据结构 | 模型 |
|---------|---------|------|
| `CLOSURE_BSDF_HAIR_REFLECTION_ID` | `HairBsdf` | 基础毛发反射 |
| `CLOSURE_BSDF_HAIR_TRANSMISSION_ID` | `HairBsdf` | 基础毛发透射 |
| `CLOSURE_BSDF_HAIR_CHIANG_ID` | `ChiangHairBSDF` | Chiang 2015 Principled Hair |
| `CLOSURE_BSDF_HAIR_HUANG_ID` | `HuangHairBSDF` | Huang 2022 微表面毛发 |

#### 次表面散射闭包 (BSSRDF)

| 闭包 ID | 数据结构 | 模型 |
|---------|---------|------|
| `CLOSURE_BSSRDF_BURLEY_ID` | `Bssrdf` | Burley 扩散近似 |
| `CLOSURE_BSSRDF_RANDOM_WALK_ID` | `Bssrdf` | 随机游走 SSS |
| `CLOSURE_BSSRDF_RANDOM_WALK_SKIN_ID` | `Bssrdf` | 随机游走皮肤 SSS |

#### 体积散射闭包 (Volume Phase Functions)

| 闭包 ID | 数据结构 | 模型 |
|---------|---------|------|
| `CLOSURE_VOLUME_HENYEY_GREENSTEIN_ID` | `HenyeyGreensteinVolume` | Henyey-Greenstein 相位函数 |
| `CLOSURE_VOLUME_RAYLEIGH_ID` | `RayleighVolume` | Rayleigh 散射 |
| `CLOSURE_VOLUME_DRAINE_ID` | `DraineVolume` | Draine 相位函数 |
| `CLOSURE_VOLUME_FOURNIER_FORAND_ID` | `FournierForandVolume` | Fournier-Forand 相位函数 |

#### 发射闭包 (Emission)

| 函数 | 说明 |
|------|------|
| `emission_setup()` | 表面发射（自发光）|
| `background_setup()` | 背景/环境光发射 |

### 微表面菲涅尔模型

`MicrofacetBsdf` 支持多种菲涅尔模型，通过 `fresnel_type` 枚举选择：

| 类型 | 结构 | 说明 |
|------|------|------|
| `NONE` | — | 无菲涅尔（常数反射率）|
| `DIELECTRIC` | — | 介电体菲涅尔（使用 IOR）|
| `DIELECTRIC_TINT` | `FresnelDielectricTint` | 带色调的介电体菲涅尔（OSL MaterialX）|
| `CONDUCTOR` | `FresnelConductor` | 导体菲涅尔（复数折射率）|
| `GENERALIZED_SCHLICK` | `FresnelGeneralizedSchlick` | 广义 Schlick 近似（F0/F90 参数）|
| `F82_TINT` | `FresnelF82Tint` | F82 色调模型（金属反射的精确拟合）|

## 模块架构

```
着色器评估 (SVM / OSL)
        │
        ▼
  bsdf_alloc() / closure_alloc()          (alloc.h)
        │
        ▼
  bsdf_*_setup()                          (各闭包头文件)
        │
        ▼
  ShaderData::closure[] 数组
        │
        ├────────────────────────────┐
        ▼                            ▼
  bsdf_sample()                   bsdf_eval()              (bsdf.h)
  ├── bsdf_diffuse_sample()       ├── bsdf_diffuse_eval()
  ├── bsdf_microfacet_ggx_sample()├── bsdf_microfacet_ggx_eval()
  ├── bsdf_hair_chiang_sample()   ├── bsdf_hair_chiang_eval()
  ├── ...                         └── ...
  │
  ▼
  路径标签 (LABEL_REFLECT | LABEL_DIFFUSE, ...)
        │
        ▼
  积分器路径追踪循环

体积路径：
  volume_phase_sample() / volume_phase_eval()   (volume.h)
  ├── volume_henyey_greenstein_*()
  ├── volume_rayleigh_*()
  ├── volume_draine_*()
  └── volume_fournier_forand_*()
```

## 内核函数入口

### 表面闭包入口

| 入口函数 | 文件 | 说明 |
|----------|------|------|
| `bsdf_sample()` | `bsdf.h` | 统一 BSDF 采样入口，switch-case 分派到具体闭包 |
| `bsdf_eval()` | `bsdf.h` | 统一 BSDF 评估入口，switch-case 分派到具体闭包 |
| `bsdf_label()` | `bsdf.h` | 根据闭包类型和出射方向返回路径标签 |
| `bsdf_blur()` | `bsdf.h` | 对闭包的粗糙度进行模糊处理（用于焦散钳制）|
| `bsdf_albedo()` | `bsdf.h` | 估算闭包的反照率（用于降噪和颜色通道）|
| `bsdf_roughness_eta()` | `bsdf.h` | 获取闭包的粗糙度和折射率 |

### 体积闭包入口

| 入口函数 | 文件 | 说明 |
|----------|------|------|
| `volume_phase_sample()` | `volume.h` | 统一体积相位函数采样入口 |
| `volume_phase_eval()` | `volume.h` | 统一体积相位函数评估入口 |
| `volume_extinction_setup()` | `volume.h` | 设置体积消光系数 |
| `volume_sample_channel()` | `volume.h` | 按通量和反照率比例采样颜色通道 |

### 调用链

```
kernel/integrator/shade_surface.h
  └── bsdf_sample() / bsdf_eval()   →  kernel/closure/bsdf.h
        └── bsdf_diffuse_sample()    →  kernel/closure/bsdf_diffuse.h
        └── bsdf_microfacet_*()      →  kernel/closure/bsdf_microfacet.h
        └── ...

kernel/integrator/shade_volume.h
  └── volume_phase_sample() / volume_phase_eval()  →  kernel/closure/volume.h
        └── volume_henyey_greenstein_*()            →  kernel/closure/volume_henyey_greenstein.h
        └── ...
```

## GPU 兼容性

### 设备兼容

所有闭包代码使用 `ccl_device` / `ccl_device_inline` / `ccl_device_forceinline` 等宏标注，确保在以下所有设备上正确编译和运行：

| 设备 | 编译路径 | 备注 |
|------|----------|------|
| CPU | C++ 编译 | 完整功能支持 |
| CUDA / OptiX | NVCC 编译 | `bsdf_eval()` 在 CUDA 上使用 `ccl_device_inline` 避免栈溢出 |
| HIP | HIPCC 编译 | 完整功能支持 |
| Metal | MSL 编译 | 完整功能支持 |
| oneAPI | SYCL 编译 | 完整功能支持 |

### 内存模型

- 闭包数据存储在 `ShaderData::closure[]` 固定大小数组中
- 每个闭包必须 `sizeof(ShaderClosure)` 以内（通过 `static_assert` 检查）
- 额外参数（如菲涅尔数据）通过 `closure_alloc_extra()` 从数组尾部分配
- 体积闭包使用独立的 `ShaderVolumeClosure` 结构

### 编译宏控制

| 宏 | 说明 |
|----|------|
| `__SVM__` | SVM 着色器虚拟机可用（启用大部分闭包）|
| `__OSL__` | OSL 着色语言可用（启用 Burley、PhongRamp、DiffuseRamp 等 OSL 专用闭包）|
| `__PRINCIPLED_HAIR__` | 启用 Principled Hair 模型（Chiang + Huang）|
| `__KERNEL_GPU__` | GPU 编译路径 |
| `__KERNEL_CUDA__` | CUDA 编译路径（影响 `bsdf_eval()` 的内联策略）|

### GPU 性能注意事项

- `bsdf_sample()` 和 `bsdf_eval()` 中的大型 switch-case 在 GPU 上可能导致寄存器压力
- `bsdf_eval()` 在非 CUDA GPU 上使用 `ccl_device` 而非内联，在 CUDA 上使用 `ccl_device_inline`
- 闭包权重截断阈值 `CLOSURE_WEIGHT_CUTOFF` 避免分配极低权重的闭包
- 体积闭包不受权重截断限制（低密度大体积仍可能有显著贡献）

## 依赖关系

### 上游依赖（本模块依赖）

| 模块 | 说明 |
|------|------|
| `kernel/types.h` | 闭包类型枚举 `ClosureType`、`ShaderClosure`、`ShaderData` 等核心类型 |
| `kernel/sample/mapping.h` | 半球采样函数（`sample_cos_hemisphere` 等）|
| `kernel/util/lookup_table.h` | 查找表工具（用于 Sheen LTC 和微表面能量补偿）|
| `kernel/util/colorspace.h` | 颜色空间转换（用于毛发吸收系数）|
| `kernel/sample/lcg.h` | 线性同余随机数生成器（用于 Huang Hair）|
| `util/math_fast.h` | 快速数学函数（`fast_acosf` 等）|
| `util/color.h` | 颜色工具函数 |

### 下游依赖（依赖本模块）

| 模块 | 说明 |
|------|------|
| `kernel/integrator/shade_surface.h` | 表面着色，调用 `bsdf_sample()` / `bsdf_eval()` |
| `kernel/integrator/shade_volume.h` | 体积着色，调用 `volume_phase_sample()` / `volume_phase_eval()` |
| `kernel/integrator/subsurface.h` | 次表面散射，使用 BSSRDF 闭包 |
| `kernel/svm/closure.h` | SVM 着色器闭包节点，调用各 `bsdf_*_setup()` |
| `kernel/osl/closures.h` | OSL 闭包桥接 |
| `kernel/integrator/mnee.h` | Manifold Next Event Estimation，使用微表面闭包 |

## 关键算法与实现细节

### GGX 微表面模型

`bsdf_microfacet.h` 实现了完整的 GGX 和 Beckmann 微表面 BSDF：
- **各向异性支持**：`alpha_x` 和 `alpha_y` 独立控制两个方向的粗糙度
- **多种菲涅尔模型**：介电体、导体（复数 IOR）、广义 Schlick、F82 Tint
- **薄膜干涉**：`FresnelThinFilm` 支持薄膜涂层的色散效果
- **能量补偿**：`energy_scale` 因子补偿单次散射微表面模型的能量损失
- **玻璃模式**：同时支持反射和折射，通过菲涅尔系数在两者间采样

### Principled Hair 模型

两种毛发模型的实现：
- **Chiang 2015**：基于圆柱体散射理论的解析模型，支持 R/TT/TRT 波瓣
- **Huang 2022**：基于微表面的毛发散射模型，使用椭圆截面和随机采样

### Bump 阴影项

`bump_shadowing_term()` 解决了使用 bump/normal mapping 时光线穿透几何体的问题：
1. 检查入射方向和着色法线是否在光滑法线的同一侧
2. 对于漫反射闭包，使用 Cook-Torrance GGX 函数进行平滑过渡
3. 基于 Conty Estevez 等人的 "A Microfacet-Based Shadowing Function to Solve the Bump Terminator Problem"

### 体积颜色通道采样

`volume_sample_channel()` 实现了 Chiang 等人 (SIGGRAPH 2016) 的技术：按通量和单次散射反照率的比例采样颜色通道，显著减少多次弹跳体积渲染的噪声。

### 自相交修正

阴影终止器偏移 (`shift_cos_in()`) 源自 Appleseed 渲染器，通过频率乘数调整入射角余弦值，避免光滑法线插值导致的阴影终止器伪影。

## 参见

- `src/kernel/svm/closure.h` — SVM 闭包节点，创建和配置闭包
- `src/kernel/integrator/shade_surface.h` — 表面着色积分器
- `src/kernel/integrator/shade_volume.h` — 体积着色积分器
- `src/kernel/sample/` — 采样函数（半球、微表面法线分布等）
- Chiang et al., "A practical and controllable hair and fur model for production path tracing", SIGGRAPH 2016
- Huang et al., "A Microfacet-based Hair Scattering Model", CGF 2022
- Zeltner et al., "Practical Multiple-Scattering Sheen Using Linearly Transformed Cosines", 2022
- Conty Estevez et al., "A Microfacet-Based Shadowing Function to Solve the Bump Terminator Problem"
- Fujii, "Improved Oren-Nayar Diffuse", 2012
