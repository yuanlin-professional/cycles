# bsdf_phong_ramp.h - Phong 渐变色高光双向散射分布函数（BSDF）实现

## 概述

本文件实现了带颜色渐变映射的 Phong 高光反射模型，仅在 OSL（Open Shading Language）模式下可用。与标准 Phong 模型不同，该模型根据高光余弦幂次值从 8 色渐变数组中插值获取颜色，从而实现随反射角度变化的着色效果。该闭包主要用于兼容 OSL 着色器中的自定义高光效果。

## 类与结构体

### `struct PhongRampBsdf`
Phong 渐变色 BSDF 数据结构，继承自 `SHADER_CLOSURE_BASE`：
- `exponent` — Phong 高光指数，控制高光的锐利程度（值越大高光越集中）
- `colors` — 指向 8 个 `float3` 颜色值数组的指针，用于渐变色映射

## 核心函数

### `bsdf_phong_ramp_get_color(colors, pos)`
颜色渐变插值函数。将位置参数 `pos`（0 到 1）映射到 8 色数组中，对相邻颜色进行线性插值。超出范围时钳位到首尾颜色。

### `bsdf_phong_ramp_setup(bsdf)`
闭包初始化函数：
1. 设置闭包类型为 `CLOSURE_BSDF_PHONG_RAMP_ID`
2. 将指数钳位到不小于 0
3. 返回 `SD_BSDF | SD_BSDF_HAS_EVAL` 标志

### `bsdf_phong_ramp_eval(sc, wi, wo, pdf)`
BSDF 求值函数：
1. 检查入射和出射方向均在法线正半球
2. 计算反射方向 `R = 2 * (N . wi) * N - wi`
3. 计算 `cosRO = dot(R, wo)` 的 `exponent` 次幂（`cosp`）
4. PDF 为 `(exponent + 1) * 0.5 / pi * cosp`
5. 使用 `cosp` 作为位置参数从颜色渐变中获取颜色，乘以 `cosNO * (exponent + 2) * 0.5 / pi * cosp`

### `phong_ramp_exponent_to_roughness(exponent)`
将 Phong 指数转换为等效粗糙度：`sqrt(1 / ((exponent + 2) / 2))`，用于 Filter Glossy 等需要粗糙度参数的功能。

### `bsdf_phong_ramp_sample(sc, Ng, wi, rand, eval, wo, pdf, sampled_roughness)`
BSDF 采样函数：
1. 计算反射方向 `R` 并构建局部坐标系
2. 按 Phong 分布采样：`cosTheta = rand.y^(1 / (exponent + 1))`
3. 将采样方向转换到全局坐标系
4. 验证采样方向在几何法线和着色法线的正确半球
5. 标签为 `LABEL_REFLECT | LABEL_GLOSSY`

## 依赖关系
- **内部头文件**:
  - `kernel/types.h` — 内核类型定义
  - `kernel/util/colorspace.h` — 颜色空间转换（`rgb_to_spectrum`）
- **被引用**:
  - `kernel/closure/bsdf.h` — 闭包统一调度入口

## 实现细节 / 关键算法

### 条件编译
整个实现被 `#ifdef __OSL__` 包裹，仅在启用 OSL 支持时编译。这意味着该闭包不可用于 SVM（Shader Virtual Machine）着色器节点。

### Phong 反射模型
采用经典 Phong 反射公式，高光强度正比于 `cos^n(R, O)`，其中 `R` 为镜面反射方向，`O` 为出射方向，`n` 为指数。PDF 采用标准化的 Phong 采样分布。

### 颜色渐变映射
将高光余弦幂次值 `cosp = cos^n(R, O)` 作为位置参数映射到 8 色渐变数组。由于 `cosp` 在 [0, 1] 范围内，高光中心（`cosp` 接近 1）使用数组末端颜色，高光边缘（`cosp` 接近 0）使用数组起始颜色，实现了随角度变化的高光着色效果。

### 粗糙度转换
`phong_ramp_exponent_to_roughness` 提供了 Phong 指数与微面元粗糙度之间的近似转换关系，使得 Filter Glossy 功能能够正确处理该闭包。

## 关联文件
- `kernel/closure/bsdf.h` — 闭包调度入口，根据 `CLOSURE_BSDF_PHONG_RAMP_ID` 调用本文件函数
- `kernel/util/colorspace.h` — 提供 `rgb_to_spectrum` 转换
