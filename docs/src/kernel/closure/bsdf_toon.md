# bsdf_toon.h - 卡通风格双向散射分布函数(BSDF)闭包

## 概述

本文件实现了 Cycles 渲染器中的卡通着色（Toon Shading）BSDF 闭包，改编自 Open Shading Language。该闭包提供漫反射卡通（Diffuse Toon）和光泽卡通（Glossy Toon）两种模式，通过 `size`（大小）和 `smooth`（平滑度）参数控制明暗过渡，产生非真实感的卡通渲染效果。

## 类与结构体

### `ToonBsdf`
卡通 BSDF 闭包的数据结构，继承自 `SHADER_CLOSURE_BASE`。

| 成员 | 类型 | 说明 |
|------|------|------|
| `size` | `float` | 高光区域的角度大小（初始化后转换为弧度） |
| `smooth` | `float` | 明暗过渡的平滑度（初始化后转换为弧度） |

编译时通过 `static_assert` 确保 `ToonBsdf` 不超过 `ShaderClosure` 的大小。

## 核心函数

### 通用函数

#### `bsdf_toon_setup_common`
将 `size` 钳制到 `[1e-5, 1.0]` 再乘以 `pi/2` 转换为弧度，`smooth` 钳制到 `[0, 1]` 再乘以 `pi/2`。返回 `SD_BSDF | SD_BSDF_HAS_EVAL` 标志。

#### `bsdf_toon_get_intensity`
```cpp
ccl_device float bsdf_toon_get_intensity(float max_angle, float smooth, float angle)
```
根据当前角度计算卡通着色强度：
- 角度 < `max_angle`：返回 1.0（全亮）
- 角度在 `[max_angle, max_angle + smooth]` 区间：线性插值过渡
- 角度 > `max_angle + smooth`：返回 0.0（全暗）

#### `bsdf_toon_get_sample_angle`
返回采样角度上限 `min(max_angle + smooth, pi/2)`。

### 漫反射卡通

#### `bsdf_diffuse_toon_setup`
设置类型为 `CLOSURE_BSDF_DIFFUSE_TOON_ID`。

#### `bsdf_diffuse_toon_eval`
基于出射方向与法线的夹角计算卡通强度。PDF 在锥形区域内均匀分布。

#### `bsdf_diffuse_toon_sample`
在法线周围的均匀锥体内采样出射方向。返回 `LABEL_REFLECT | LABEL_DIFFUSE`。

### 光泽卡通

#### `bsdf_glossy_toon_setup`
设置类型为 `CLOSURE_BSDF_GLOSSY_TOON_ID`。

#### `bsdf_glossy_toon_eval`
先计算镜面反射方向 `R = 2*(N . wi)*N - wi`，然后基于出射方向与反射方向的夹角计算卡通强度。

#### `bsdf_glossy_toon_sample`
在镜面反射方向周围的均匀锥体内采样出射方向。返回 `LABEL_GLOSSY | LABEL_REFLECT`。

## 依赖关系
- **内部头文件**:
  - `kernel/types.h` — 内核类型定义
  - `kernel/sample/mapping.h` — 采样映射工具（`sample_uniform_cone`）
- **被引用**:
  - `src/kernel/closure/bsdf.h` — BSDF 调度总入口

## 实现细节 / 关键算法

### 卡通着色模型
卡通着色的核心是将连续的光照变化离散化为明锐的明暗分界：

1. **漫反射卡通**：以法线 `N` 为中心，在角度 `max_angle` 内为全亮，`smooth` 区间内线性过渡到全暗
2. **光泽卡通**：以镜面反射方向 `R` 为中心，使用相同的角度/平滑逻辑

### 采样策略
两种模式均使用 `sample_uniform_cone` 在锥形区域内均匀采样：
- 漫反射模式：锥体轴为法线 `N`
- 光泽模式：锥体轴为反射方向 `R`

采样后检查出射方向是否位于几何法线 `Ng` 的正确半球，不合法则返回零贡献。

## 关联文件
- `src/kernel/closure/bsdf.h` — BSDF 统一调度
- `src/kernel/closure/bsdf_diffuse.h` — 标准漫反射 BSDF（对比参考）
- `src/kernel/sample/mapping.h` — 均匀锥体采样实现
