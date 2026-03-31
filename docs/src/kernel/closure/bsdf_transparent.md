# bsdf_transparent.h - 透明双向散射分布函数(BSDF)闭包

## 概述

本文件实现了 Cycles 渲染器中的透明（Transparent）BSDF 闭包，改编自 Open Shading Language。透明 BSDF 是一种理想的直线透射闭包，光线直接穿过表面而不发生偏折。该闭包的 eval 始终返回零（delta 分布），采样时输出方向恰好为入射方向的反方向，用于实现纯透明效果（如透明阴影、Alpha 通道等）。

## 类与结构体

透明 BSDF 不需要额外的数据结构，直接使用基础的 `ShaderClosure`。

## 核心函数

### `bsdf_transparent_setup`
```cpp
ccl_device void bsdf_transparent_setup(ccl_private ShaderData *sd,
                                       const Spectrum weight,
                                       const uint32_t path_flag)
```
初始化透明 BSDF。主要流程：
1. 检查权重是否超过 `CLOSURE_WEIGHT_CUTOFF` 截止阈值
2. 将权重累加到 `sd->closure_transparent_extinction`
3. 若已存在透明 BSDF（`SD_TRANSPARENT` 标志已设置），则将权重累加到现有闭包上，避免重复分配
4. 否则创建新的透明 BSDF，设置 `SD_BSDF | SD_TRANSPARENT` 标志
5. 特殊处理 `PATH_RAY_TERMINATE`：当路径终止时，临时增加 `num_closure_left` 以确保仍能分配透明闭包

### `bsdf_transparent_eval`
```cpp
ccl_device Spectrum bsdf_transparent_eval(...)
```
始终返回零光谱值和零 PDF。透明 BSDF 是 delta 分布，不参与常规求值。

### `bsdf_transparent_sample`
```cpp
ccl_device int bsdf_transparent_sample(const ccl_private ShaderClosure *sc,
                                       const float3 Ng,
                                       const float3 wi,
                                       ccl_private Spectrum *eval,
                                       ccl_private float3 *wo,
                                       ccl_private float *pdf)
```
采样函数：
- 输出方向 `wo = -wi`（直线透射）
- PDF 和 eval 均设置为极大值 `1e6`，用于在多重重要性采样（MIS）中赋予高权重
- 返回标签 `LABEL_TRANSMIT | LABEL_TRANSPARENT`

## 依赖关系
- **内部头文件**:
  - `kernel/types.h` — 内核类型定义
  - `kernel/closure/alloc.h` — 闭包内存分配
- **被引用**:
  - `src/kernel/closure/bsdf.h` — BSDF 调度总入口
  - `src/kernel/closure/bsdf_principled_hair_huang.h` — Huang 毛发模型中的透明处理

## 实现细节 / 关键算法

### 闭包合并优化
为避免多次分配透明闭包浪费内存，当着色点上已存在透明 BSDF 时（检测 `SD_TRANSPARENT` 标志），新的透明权重直接累加到已有闭包上。这种优化在有多层透明混合时尤为重要。

### 路径终止特殊处理
当 `path_flag` 包含 `PATH_RAY_TERMINATE` 时，说明路径已被终止（所有闭包数设为零）。此时临时将 `num_closure_left` 设为 1，以便仍能分配透明闭包。如果分配失败，则恢复为 0。这确保了即使在路径终止的情况下，透明度信息仍能正确传播。

### MIS 权重策略
采样时 PDF 和 eval 被设置为极大值 `1e6`，而非理论上的无穷大。这是一种实用的近似，在 MIS 计算中能有效保证 delta 分布的采样路径获得正确的高权重。

## 关联文件
- `src/kernel/closure/bsdf.h` — BSDF 统一调度
- `src/kernel/closure/alloc.h` — 闭包分配器
- `src/kernel/closure/bsdf_ray_portal.h` — 光线传送门，类似的透明消光处理
