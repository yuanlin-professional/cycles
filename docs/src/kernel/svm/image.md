# image.h - 图像纹理节点的SVM实现

## 概述

`image.h` 实现了着色器虚拟机(SVM)中的图像纹理相关节点，包括普通图像纹理、盒式投影(Box Projection)纹理和环境纹理。这些节点负责从内核中的纹理图像数据中采样颜色值，支持多种投影方式（平面、球面、管状、盒式）、UDIM 多瓦片纹理、以及 sRGB 解压缩和预乘 Alpha 处理。

## 核心函数

### `svm_image_texture`
- **签名**: `ccl_device float4 svm_image_texture(KernelGlobals kg, const int id, const float x, float y, const uint flags)`
- **功能**: 底层纹理采样函数。
- **流程**:
  1. 如果 `id == -1`（无效纹理），返回缺失纹理的默认颜色
  2. 调用 `kernel_tex_image_interp` 执行双线性/双三次插值采样
  3. 如果设置 `NODE_IMAGE_ALPHA_UNASSOCIATE`，将预乘 Alpha 转换为非预乘（`r /= alpha`）
  4. 如果设置 `NODE_IMAGE_COMPRESS_AS_SRGB`，执行 sRGB 到线性色彩空间转换

### `texco_remap_square`
- **功能**: 将 [0,1] 范围的坐标重映射到 [-1,1] 范围，用于球面/管状投影前的坐标预处理。

### `svm_node_tex_image`
- **签名**: `ccl_device_noinline int svm_node_tex_image(KernelGlobals kg, ShaderData *sd, float *stack, const uint4 node, int offset)`
- **功能**: 图像纹理节点主函数。
- **投影模式**:
  - `NODE_IMAGE_PROJ_SPHERE` — 球面投影：先重映射坐标，再用 `map_to_sphere` 转换
  - `NODE_IMAGE_PROJ_TUBE` — 管状投影：先重映射坐标，再用 `map_to_tube` 转换
  - 默认 — 平面投影：直接使用 `(co.x, co.y)` 作为 UV
- **UDIM 瓦片支持**:
  1. 根据 UV 整数部分确定瓦片编号 `tile = 1001 + 10*ty + tx`
  2. 遍历后续节点数据查找匹配的瓦片 ID
  3. 找到后将 UV 偏移到瓦片内相对坐标
- **输出**: 颜色(float3) 和 alpha(float) 分别存入栈

### `svm_node_tex_image_box`
- **签名**: `ccl_device_noinline void svm_node_tex_image_box(KernelGlobals kg, ShaderData *sd, float *stack, const uint4 node)`
- **功能**: 盒式投影纹理（三平面投影）。
- **算法**:
  1. 获取物体空间法线并取绝对值，归一化为重心坐标
  2. 根据 `blend` 参数将空间划分为 7 个区域：3 个角区（单纹理）、3 个边区（两纹理混合）、1 个中心区（三纹理混合）
  3. 对每个权重非零的轴，使用对应平面的 UV 坐标采样纹理
  4. 累加加权结果（翻转方向避免纹理镜像）

### `svm_node_tex_environment`
- **签名**: `ccl_device_noinline void svm_node_tex_environment(KernelGlobals kg, ShaderData *sd, float *stack, const uint4 node)`
- **功能**: 环境纹理节点。
- **投影模式**:
  - 等距矩形投影(Equirectangular): `direction_to_equirectangular(co)`
  - 镜像球投影(Mirror Ball): `direction_to_mirrorball(co)`

## 依赖关系

- **内部头文件**:
  - `kernel/globals.h` — 全局内核数据
  - `kernel/image.h` — 纹理图像采样（`kernel_tex_image_interp`）
  - `kernel/camera/projection.h` — 投影映射函数（球面、管状、等距矩形、镜像球）
  - `kernel/geom/object.h` — 物体空间法线变换
  - `kernel/svm/util.h` — SVM 栈操作工具
  - `util/color.h` — sRGB 色彩空间转换
- **被引用**: `kernel/svm/svm.h`

## 实现细节 / 关键算法

1. **UDIM 瓦片查找**: UDIM 编号遵循标准约定 `1001 + 10*row + col`，列范围 [0,9]，行不限。每个 SVM 节点可打包两个瓦片(tile_node.x/y 和 tile_node.z/w)，减少节点数据量。

2. **盒式投影混合**: 混合算法基于等边三角形的重心坐标思想。`blend` 参数控制混合区域的宽度。`limit = 0.5 * (1 + blend)` 定义了单纹理区域的边界。当 `blend = 0` 时，使用硬边界（无混合），回退到单轴。

3. **纹理方向翻转**: 盒式投影中对 UV 坐标进行了条件翻转（如 `signed_N.x < 0` 时翻转 `co.y`），确保在法线方向变化时纹理不会产生镜像接缝。

4. **Alpha 处理**: `NODE_IMAGE_ALPHA_UNASSOCIATE` 标志将预乘 Alpha 转换为非关联 Alpha（直接 Alpha）。这在将纹理 Alpha 作为独立因子使用时很重要。

## 关联文件

- `kernel/image.h` — 底层纹理图像采样实现
- `kernel/camera/projection.h` — 各种投影映射
- `kernel/svm/svm.h` — SVM 主调度器
