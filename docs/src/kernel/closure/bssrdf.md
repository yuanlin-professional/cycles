# bssrdf.h - 次表面散射(BSSRDF)闭包

## 概述

本文件实现了 Cycles 渲染器中的次表面散射（BSSRDF）闭包，用于模拟光线进入半透明材质内部并在不同位置射出的效果，如皮肤、蜡烛、大理石等。该文件支持多种次表面散射模型：Christensen-Burley 模型（基于 Pixar 近似反射率剖面）、随机游走模型（Random Walk）及随机游走皮肤模型。当散射半径过小时自动降级为漫反射 BSDF。

## 类与结构体

### `Bssrdf`
次表面散射闭包的数据结构，继承自 `SHADER_CLOSURE_BASE`。

| 成员 | 类型 | 说明 |
|------|------|------|
| `radius` | `Spectrum` | 散射半径（各光谱通道独立控制散射距离） |
| `albedo` | `Spectrum` | 表面反照率 |
| `anisotropy` | `float` | 散射各向异性参数 |
| `ior` | `float` | 折射率（控制折射入射弹跳） |
| `alpha` | `float` | 粗糙度参数（用于随机游走皮肤模型） |

编译时通过 `static_assert` 确保 `Bssrdf` 不超过 `ShaderClosure` 的大小。

## 核心函数

### 偶极子模型辅助函数

#### `bssrdf_dipole_compute_Rd`
```cpp
ccl_device float bssrdf_dipole_compute_Rd(float alpha_prime, float fourthirdA)
```
计算偶极子扩散反射率 Rd，用于将反照率转换为扩散参数。

#### `bssrdf_dipole_compute_alpha_prime`
```cpp
ccl_device float bssrdf_dipole_compute_alpha_prime(float rd, float fourthirdA)
```
使用二分法（12 次迭代）从目标扩散反射率 rd 反求 alpha_prime。

### 半径设置

#### `bssrdf_setup_radius`
```cpp
ccl_device void bssrdf_setup_radius(ccl_private Bssrdf *bssrdf, ClosureType type)
```
根据模型类型调整散射半径：
- **Burley / Random Walk**：乘以 `0.25 / pi` 缩放平均自由程
- **Random Walk Skin**：基于折射率和反照率通过偶极子模型调整半径

### Christensen-Burley 模型

#### `bssrdf_burley_setup`
初始化 Burley 模型：计算兼容的平均自由程，并使用 `bssrdf_burley_fitting` 拟合曲线缩放半径。

#### `bssrdf_burley_eval`
Burley 反射率剖面求值（论文公式 3），归一化漫反射模型：
```
(exp(-r/d) + exp(-r/(3d))) / (4d)
```

#### `bssrdf_burley_pdf`
基于 `bssrdf_burley_eval` 的概率密度函数，除以截断 CDF 值进行归一化。

#### `bssrdf_burley_root_find`
Newton-Raphson 法求解 CDF 的反函数，用于重要性采样。初始猜测基于手动曲线拟合以加速收敛（通常 4 次内收敛）。

#### `bssrdf_burley_sample`
采样函数：通过 `bssrdf_burley_root_find` 获取采样半径 `r`，并计算对应的探测高度 `h`。

### 光谱通道采样

#### `bssrdf_num_channels`
统计半径大于零的有效光谱通道数。

#### `bssrdf_sample`
按通道均匀采样，随机选择一个有效通道后调用 `bssrdf_burley_sample`。

#### `bssrdf_eval` / `bssrdf_pdf`
逐通道求值/计算 PDF，PDF 为所有通道的平均。

### 分配与初始化

#### `bssrdf_alloc`
分配次表面散射闭包内存，检查权重截止。

#### `bssrdf_setup`
```cpp
ccl_device int bssrdf_setup(ccl_private ShaderData *sd,
                            ccl_private Bssrdf *bssrdf,
                            int path_flag, ClosureType type)
```
完整的初始化流程：
1. 钳制各向异性 `[0, 0.9]` 和折射率 `[1.01, 3.8]`
2. 对 Random Walk Skin 模型固定 `alpha = 1.0`
3. 检查各通道半径是否足够大（>= `BSSRDF_MIN_RADIUS`）
4. 半径过小的通道降级为漫反射 BSDF（`bsdf_diffuse_setup`）
5. 对 Burley 模型在漫反射祖先路径后全部降级为漫反射，以减少噪声
6. 调用 `bssrdf_setup_radius` 调整半径
7. 设置 `SD_BSSRDF` 标志

## 依赖关系
- **内部头文件**:
  - `kernel/types.h` — 内核类型定义
  - `kernel/closure/alloc.h` — 闭包内存分配
  - `kernel/closure/bsdf_diffuse.h` — 漫反射 BSDF（小半径降级使用）
- **被引用**:
  - `src/kernel/svm/closure.h` — SVM 闭包节点
  - `src/kernel/integrator/subsurface_disk.h` — 盘采样次表面散射积分器
  - `src/kernel/integrator/subsurface.h` — 次表面散射积分器入口
  - `src/kernel/osl/closures_setup.h` — OSL 闭包初始化

## 实现细节 / 关键算法

### Christensen-Burley 反射率剖面
基于 Pixar 2015 年论文，使用两项指数之和近似扩散反射率剖面。截断半径为 `16d`（对应 CDF 约 99.64%），在保证精度的同时提高采样效率。

### 偶极子 alpha_prime 反求
使用二分法而非 Newton-Raphson，保证在 `[0, 1]` 范围内稳定收敛。12 次迭代提供约 `2^-12 ~ 2.4e-4` 的精度。

### 通道降级策略
当某个光谱通道的散射半径低于 `BSSRDF_MIN_RADIUS` 时，该通道的权重转移到漫反射 BSDF。这避免了极短散射距离下的数值精度问题和采样效率低下。

### Burley 模型在漫反射路径后的降级
在漫反射祖先路径（`PATH_RAY_DIFFUSE_ANCESTOR`）后，Christensen-Burley 模型完全降级为漫反射。原因是在深层路径中次表面散射效果几乎不可见，但盘采样的低命中率会产生大量噪声。

## 关联文件
- `src/kernel/closure/bsdf_diffuse.h` — 漫反射 BSDF，用于小半径降级
- `src/kernel/integrator/subsurface.h` — 次表面散射积分器
- `src/kernel/integrator/subsurface_disk.h` — 盘采样积分方法
- `src/kernel/integrator/subsurface_random_walk.h` — 随机游走积分方法
