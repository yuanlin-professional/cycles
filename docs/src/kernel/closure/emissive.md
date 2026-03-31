# emissive.h - 发光（自发射）与背景闭包

## 概述

本文件实现了 Cycles 渲染器中的发光（Emission）和背景（Background）闭包，改编自 Open Shading Language。这两种闭包均用于处理表面自发光，区别仅在于语义用途。发光闭包通常用于物体表面的自发射光源，背景闭包用于场景背景光照。两者共享相同的底层实现。

## 类与结构体

发光和背景闭包不需要额外的数据结构。发光信息直接存储在 `ShaderData` 的 `closure_emission_background` 成员中。

## 核心函数

### `background_setup`
```cpp
ccl_device void background_setup(ccl_private ShaderData *sd, const Spectrum weight)
```
初始化背景闭包。若 `SD_EMISSION` 标志已设置，则累加权重；否则设置标志并初始化发光光谱值。

### `emission_setup`
```cpp
ccl_device void emission_setup(ccl_private ShaderData *sd, const Spectrum weight)
```
初始化发光闭包。实现与 `background_setup` 完全相同：若已有发光标志则累加权重，否则设置标志和初始值。

### `emissive_pdf`
```cpp
ccl_device float emissive_pdf(const float3 Ng, const float3 wi)
```
计算发光表面在 `wi` 方向上的概率密度。若入射方向与几何法线夹角余弦大于零（面向光源），返回 1.0；否则返回 0.0。这是一个恒定的均匀 PDF（面光源均匀发光）。

### `emissive_sample`
```cpp
ccl_device void emissive_sample(...)
```
发光采样函数，**当前未实现**（标注为 TODO）。实际的光源采样通过光源采样系统完成，而非此函数。

### `emissive_simple_eval`
```cpp
ccl_device Spectrum emissive_simple_eval(const float3 Ng, const float3 wi)
```
简单发光求值。调用 `emissive_pdf` 获取概率密度，将其转换为光谱值返回。

## 依赖关系
- **内部头文件**:
  - `kernel/types.h` — 内核类型定义
- **被引用**:
  - `src/kernel/svm/closure.h` — SVM 闭包节点
  - `src/kernel/integrator/surface_shader.h` — 表面着色器积分
  - `src/kernel/osl/closures_setup.h` — OSL 闭包初始化

## 实现细节 / 关键算法

### 发光闭包的简化设计
与 BSDF 闭包不同，发光闭包不需要在闭包数组中分配独立条目。所有发光权重直接累加到 `ShaderData::closure_emission_background` 中，这是一种高效的设计，因为发光不需要采样方向（光源采样由独立系统处理）。

### PDF 模型
发光表面假设为均匀的朗伯发射体——在面向观察者的方向上返回恒定的 PDF = 1.0，背面为 0。注意此处使用了 `fabsf(dot(Ng, wi))`，取绝对值意味着双面发光表面也能正确处理。

### 背景与发光的统一实现
`background_setup` 和 `emission_setup` 的代码完全一致。语义区分在上层处理：背景闭包参与环境光照计算，发光闭包参与直接光照和间接光照。

## 关联文件
- `src/kernel/integrator/surface_shader.h` — 表面着色器评估，调用 `emissive_simple_eval`
- `src/kernel/svm/closure.h` — SVM 闭包节点中设置发光和背景
- `src/kernel/light/light.h` — 光源采样系统（与发光闭包协同工作）
