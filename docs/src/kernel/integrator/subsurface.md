# subsurface.h - 次表面散射入口与调度

## 概述

`subsurface.h` 是 Cycles 次表面散射(SSS)系统的主入口文件，协调从表面着色到次表面散射模拟的过渡。该文件负责设置次表面散射弹射（包括入射折射采样）、在散射完成后设置出口点的漫反射BSDF，以及根据散射类型（随机游走或圆盘采样）调度对应的算法实现。

## 核心函数

### `subsurface_entry_bounce`
计算次表面散射的入射弹射方向。对于随机游走皮肤模型（`CLOSURE_BSSRDF_RANDOM_WALK_SKIN_ID`），有50%概率采样漫反射入射或粗糙折射入射。折射入射使用 GGX 微面元VNDF采样确定法线，然后通过 Snell 定律计算折射方向。

### `subsurface_bounce`
设置次表面散射弹射的完整路径状态：
- 写入光线起点和方向到积分器状态
- 计算并应用包含菲涅尔权重的采样权重
- 根据 BSSRDF 类型设置路径标志（`PATH_RAY_SUBSURFACE_DISK` 或 `PATH_RAY_SUBSURFACE_RANDOM_WALK`）
- 将 BSSRDF 参数（`albedo`, `radius`, `anisotropy`）写入状态
- 返回 `LABEL_SUBSURFACE_SCATTER` 标签

### `subsurface_shader_data_setup`
在次表面散射出口点设置漫反射 BSDF 闭包。清除现有闭包，分配一个 `DiffuseBsdf`，使用经过凹凸贴图修正的法线。此函数替代了出口点的完整着色器评估。

### `subsurface_scatter`
执行实际的次表面散射模拟。根据路径标志选择随机游走(`subsurface_random_walk`)或圆盘采样(`subsurface_disk`)方法。散射完成后：
- 更新体积栈（如果对象与体积相交）
- 反转光线方向使其"从外部"射向出口点
- 将交点和光线写入积分器状态
- 根据着色器标志调度到适当的表面着色内核（MNEE / Raytrace / 普通）

## 依赖关系

- **内部头文件**:
  - `kernel/closure/alloc.h` - 闭包内存分配
  - `kernel/closure/bsdf_diffuse.h` - 漫反射 BSDF
  - `kernel/closure/bssrdf.h` - BSSRDF 闭包
  - `kernel/integrator/intersect_volume_stack.h` - 体积栈更新
  - `kernel/integrator/path_state.h` - 路径状态
  - `kernel/integrator/subsurface_disk.h` - 圆盘采样实现
  - `kernel/integrator/subsurface_random_walk.h` - 随机游走实现
  - `kernel/integrator/surface_shader.h` - 表面着色器工具
- **被引用**: `shade_surface.h`（表面着色中调用 `subsurface_bounce`）、`intersect_subsurface.h`（调用 `subsurface_scatter`）

## 实现细节 / 关键算法

1. **入射折射采样**: 使用 GGX 微面元 VNDF（可见法线分布函数）采样获取微面元法线，通过 Snell 定律计算折射方向。代码注释说明跳过了标准的 BSDF/PDF 权重计算（`(1 + lambdaI) / (1 + lambdaI + lambdaO)`），因为精确权重会导致暗化，而此处只关心方向分布。

2. **皮肤模型的混合入射**: `RANDOM_WALK_SKIN` 模型以50%概率在漫反射和固定粗糙度为1.0的折射入射之间选择，更好地模拟皮肤的光线入射特性。

3. **出口点处理**: 散射后光线位置在出口点表面，但方向指向内部。通过将光线位置偏移 `2 * tmax` 并反转方向，模拟从外部射入的光线，确保法线朝向正确（前/后面判断）。

4. **着色器变体调度**: 出口点使用排序调度（`integrator_path_next_sorted`）选择着色内核，考虑焦散（MNEE）和光线追踪节点需求，确保缓存一致性。

## 关联文件

- `subsurface_disk.h` - 圆盘采样(Burley模型)的具体实现
- `subsurface_random_walk.h` - 随机游走散射的具体实现
- `shade_surface.h` - 在表面着色流程中触发次表面散射
- `intersect_subsurface.h` - 次表面散射交点内核
