# stdcycles.h - Cycles 标准开放着色语言头文件

## 概述

`stdcycles.h` 是 Cycles 渲染器开放着色语言（OSL）着色器系统的核心头文件，几乎所有 OSL 着色器文件都会包含此头文件。它在标准 OSL 头文件 `stdosl.h` 的基础上，声明了 Cycles 特有的内建闭包（closure）和辅助函数。该文件源自 Sony Pictures Imageworks 的开放着色语言项目，经 Blender Foundation 适配。

## 宏定义/函数/类型

### 常量定义

| 宏名 | 值 | 说明 |
|------|-----|------|
| `FLT_MAX` | 3.402823466e+38 | 单精度浮点数最大值 |
| `BUMP_FILTER_WIDTH` | 0.1 | 凹凸贴图坐标评估的默认偏移量，单位为像素 |

### 内建标识宏

| 宏名 | 说明 |
|------|------|
| `BUILTIN` | 标记内建闭包声明 `[[int builtin = 1]]` |
| `BUILTIN_DERIV` | 标记支持导数的内建闭包声明 `[[int builtin = 1, int deriv = 1]]` |

### 内建闭包声明

#### 漫反射与特殊 BSDF
- `diffuse_ramp(normal N, color colors[8])` - 带颜色渐变的漫反射
- `phong_ramp(normal N, float exponent, color colors[8])` - 带颜色渐变的 Phong 高光
- `diffuse_toon(normal N, float size, float smooth)` - 卡通漫反射
- `glossy_toon(normal N, float size, float smooth)` - 卡通高光
- `ashikhmin_velvet(normal N, float sigma)` - Ashikhmin 天鹅绒
- `sheen(normal N, float roughness)` - 光泽（布料）
- `ambient_occlusion()` - 环境光遮蔽

#### 微面元模型
- `microfacet_f82_tint(...)` - 带 F82 色调的微面元 BSDF
- `microfacet_multi_ggx_glass(normal N, float ag, float eta, color C)` - 多重散射 GGX 玻璃
- `microfacet_multi_ggx_aniso(normal N, vector T, float ax, float ay, color C)` - 多重散射各向异性 GGX

#### 次表面散射
- `bssrdf(string method, normal N, vector radius, color albedo)` - BSSRDF 次表面散射

#### 毛发
- `hair_reflection(...)` - 毛发反射
- `hair_transmission(...)` - 毛发透射
- `hair_chiang(...)` - Chiang 毛发模型
- `hair_huang(...)` - Huang 毛发模型

#### 体积
- `henyey_greenstein(float g)` - Henyey-Greenstein 相位函数
- `fournier_forand(float B, float IOR)` - Fournier-Forand 相位函数
- `draine(float g, float alpha)` - Draine 相位函数
- `rayleigh()` - Rayleigh 散射
- `absorption()` - 体积吸收

#### 光线传送门
- `ray_portal_bsdf(vector position, vector direction)` - 光线传送门 BSDF

### 辅助函数

- `camera_shader_raster_position()` - 返回相机着色器的光栅位置（返回 `P`）
- `camera_shader_random_sample()` - 返回相机着色器的随机采样（返回 `vector(N)`）

## 依赖关系

- **上游依赖**：`stdosl.h`（OSL 标准库头文件）
- **被依赖**：几乎所有 Cycles OSL 着色器文件均通过 `#include "stdcycles.h"` 引入此头文件
- **头文件保护**：使用 `CCL_STDCYCLESOSL_H` 宏防止重复包含
