# displace.h - 置换与凹凸节点的着色器虚拟机实现

## 概述

本文件实现了着色器虚拟机（SVM）中的凹凸（Bump）节点、置换（Displacement）节点和向量置换（Vector Displacement）节点。凹凸节点通过高度图的有限差分计算扰动法线；置换节点沿法线方向偏移表面位置；向量置换节点在指定空间中使用三维向量进行完整的表面位移。

## 核心函数

### `svm_node_set_bump`

```c
template<uint node_feature_mask>
ccl_device_noinline int svm_node_set_bump(
    KernelGlobals kg,
    ccl_private ShaderData *sd,
    ccl_private float *stack,
    const uint4 node,
    int offset)
```

**功能**：根据高度图的采样值计算扰动后的法线（凹凸贴图）。

**参数**：
- `node.y`：打包的 normal_offset、scale_offset、invert、use_object_space
- `node.z`：打包的 c_offset（中心高度）、x_offset（X 偏移高度）、y_offset（Y 偏移高度）、strength_offset
- `node.w`：打包的 out_offset 和 bump_state_offset

**执行流程**：
1. 读取法线输入、缩放系数、反转标志和物体空间标志
2. 获取位置微分 dP（从 bump_state 或紧凑形式）
3. 计算表面切线 `Rx = cross(dP.dy, N)` 和 `Ry = cross(N, dP.dx)`
4. 读取三个高度采样值（中心 h_c、X 偏移 h_x、Y 偏移 h_y）
5. 计算表面梯度 `surfgrad = (h_x - h_c) * Rx + (h_y - h_c) * Ry`
6. 计算扰动法线 `N' = normalize(filter_width * |det| * N - scale * sign(det) * surfgrad)`
7. 与原始法线按 strength 进行混合

### `svm_node_set_displacement`

```c
template<uint node_feature_mask>
ccl_device void svm_node_set_displacement(
    ccl_private ShaderData *sd,
    ccl_private float *stack,
    const uint fac_offset)
```

**功能**：将置换向量累加到着色点位置 `sd->P`。

### `svm_node_displacement`

```c
template<uint node_feature_mask>
ccl_device_noinline void svm_node_displacement(
    KernelGlobals kg,
    ccl_private ShaderData *sd,
    ccl_private float *stack,
    const uint4 node)
```

**功能**：根据高度值、中间级别和缩放系数计算沿法线方向的标量置换。

**参数**：
- `height`：高度值
- `midlevel`：中间级别（零位偏移）
- `scale`：缩放系数
- `normal`：置换方向（默认为着色法线）
- `space`：物体空间或世界空间

**公式**：`dP = normal * (height - midlevel) * scale`

### `svm_node_vector_displacement`

```c
template<uint node_feature_mask>
ccl_device_noinline int svm_node_vector_displacement(
    KernelGlobals kg,
    ccl_private ShaderData *sd,
    ccl_private float *stack,
    const uint4 node,
    int offset)
```

**功能**：使用三维向量（而非标量高度）执行置换，支持切线空间、物体空间和世界空间。

**切线空间处理流程**：
1. 获取物体空间法线
2. 查找切线属性和切线符号属性
3. 构建 TBN 矩阵（切线、副切线、法线）
4. 将置换向量从切线空间变换到物体空间
5. 再变换到世界空间

## 依赖关系

- **内部头文件**：`kernel/geom/attribute.h`、`kernel/geom/object.h`、`kernel/geom/primitive.h`、`kernel/svm/util.h`、`kernel/util/differential.h`
- **被引用**：`kernel/svm/svm.h`（SVM 主调度器）

## 实现细节 / 关键算法

- **凹凸贴图的数学原理**：`svm_node_set_bump` 使用表面梯度法（Surface Gradient Method）计算扰动法线。核心思想是：
  - 对高度函数 h 在表面上的两个微分方向（dP/dx, dP/dy）进行有限差分采样
  - 构造辅助切线 `Rx = dP/dy x N` 和 `Ry = N x dP/dx`
  - 表面梯度为 `surfgrad = dh/dx * Rx + dh/dy * Ry`
  - 扰动法线为 `N' = det * N - scale * surfgrad`，其中 `det = dP/dx . Rx`

- **模板参数 `node_feature_mask`**：用于编译期特性裁剪，通过 `IF_KERNEL_NODES_FEATURE(BUMP)` 宏在不需要凹凸/置换功能的内核变体中消除相关代码。

- **物体空间凹凸**：当 `use_object_space` 为真时，法线和微分先变换到物体空间计算，再变换回世界空间。这确保了在物体变形动画中凹凸效果的稳定性。

- **滤波宽度**：`bump_filter_width` 控制有限差分采样的间距，影响凹凸贴图的细节层级和抗锯齿效果。

- **置换空间**：标量置换支持物体空间和世界空间；向量置换额外支持切线空间，切线空间通过 TBN 矩阵实现。

## 关联文件

- `kernel/svm/bump.h`：凹凸评估的进入/退出节点（保存和恢复着色器状态）
- `kernel/svm/svm.h`：SVM 主调度入口
- `kernel/geom/attribute.h`：属性查找（切线、符号等）
- `kernel/geom/object.h`：物体空间变换
- `kernel/util/differential.h`：微分工具函数
