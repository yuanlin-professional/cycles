# nodes.h - BVH 节点光线交叉测试函数

## 概述

`nodes.h` 实现了 BVH2 二叉树内部节点的光线-包围盒交叉测试函数。该文件是 BVH 遍历的核心数学运算层，提供了三种节点交叉检测方式：轴对齐包围盒（AABB）快速测试、非轴对齐包围盒（OBB，用于毛发曲线等非轴对齐几何体）测试，以及一个自动分发的统一入口。所有函数均标记为 `ccl_device_forceinline` 以确保最大内联性能。

## 类与结构体

本文件不定义新结构体，但使用 `Transform` 结构体表示 3x4 变换矩阵（用于非轴对齐节点的空间变换）。

## 枚举与常量

| 常量 | 说明 |
|---|---|
| `PATH_RAY_NODE_UNALIGNED` | 节点标志位，标识该节点使用非轴对齐包围盒 |
| `__VISIBILITY_FLAG__` | 编译开关，启用可见性标志检测（约 5% 性能开销） |

## 核心函数

### bvh_unaligned_node_fetch_space()
- **签名**: `ccl_device_forceinline Transform bvh_unaligned_node_fetch_space(KernelGlobals kg, const int node_addr, const int child)`
- **功能**: 从 BVH 节点数据中获取非轴对齐子节点的空间变换矩阵。每个子节点占用 3 行 `float4` 数据（存储在 `bvh_nodes` 数组中偏移 `node_addr + child * 3` 处），组成一个 3x4 变换矩阵。

### bvh_aligned_node_intersect()
- **签名**: `ccl_device_forceinline int bvh_aligned_node_intersect(KernelGlobals kg, const float3 P, const float3 idir, const float tmin, const float tmax, const int node_addr, const uint visibility, float dist[2])`
- **功能**: 对 BVH 节点的两个轴对齐子包围盒进行光线交叉测试。使用经典的 slab 方法（光线与 AABB 各轴平面交叉）计算 `tmin`/`tmax` 区间重叠。返回值为 2 位掩码：bit 0 表示子节点 0 被命中，bit 1 表示子节点 1 被命中。`dist[0]`/`dist[1]` 输出各子节点的最近交叉距离，用于遍历排序。若启用 `__VISIBILITY_FLAG__`，还会检查节点可见性标志。

### bvh_unaligned_node_intersect_child()
- **签名**: `ccl_device_forceinline bool bvh_unaligned_node_intersect_child(KernelGlobals kg, const float3 P, const float3 dir, const float tmin, const float tmax, const int node_addr, const int child, float dist[2])`
- **功能**: 对单个非轴对齐子节点进行光线交叉测试。首先获取子节点的空间变换矩阵，将光线变换到节点局部空间（单位立方体），然后使用 slab 方法检测交叉。返回是否命中。

### bvh_unaligned_node_intersect()
- **签名**: `ccl_device_forceinline int bvh_unaligned_node_intersect(KernelGlobals kg, const float3 P, const float3 dir, const float tmin, const float tmax, const int node_addr, const uint visibility, float dist[2])`
- **功能**: 对 BVH 节点的两个非轴对齐子包围盒进行光线交叉测试。依次调用 `bvh_unaligned_node_intersect_child()` 检测两个子节点，返回与 `bvh_aligned_node_intersect()` 相同格式的 2 位掩码。

### bvh_node_intersect()
- **签名**: `ccl_device_forceinline int bvh_node_intersect(KernelGlobals kg, const float3 P, const float3 dir, const float3 idir, const float tmin, const float tmax, const int node_addr, const uint visibility, float dist[2])`
- **功能**: 统一的节点交叉分发函数。检查节点的 `PATH_RAY_NODE_UNALIGNED` 标志位：若为非轴对齐节点则调用 `bvh_unaligned_node_intersect()`（需要 `dir` 参数），否则调用 `bvh_aligned_node_intersect()`（使用预计算的 `idir` 逆方向）。

## 依赖关系

### 内部头文件
- `kernel/geom/object.h` - 物体几何变换函数（`transform_direction`、`transform_point`）
- `kernel/globals.h` - 内核全局数据访问（`kernel_data_fetch`）

### 被引用
- `src/kernel/bvh/bvh.h` - 通过 `#include "kernel/bvh/nodes.h"` 直接引入

## 实现细节 / 关键算法

### Slab 方法（轴对齐包围盒交叉）

`bvh_aligned_node_intersect()` 使用经典的光线-AABB slab 交叉算法：

1. BVH 节点数据布局：`node0`/`node1`/`node2` 分别存储 X/Y/Z 轴的包围盒边界（`.x`/`.y` 为两个子节点的最小值，`.z`/`.w` 为最大值）
2. 计算光线与各轴 slab 的交叉区间：`(boundary - P) * idir`
3. 取所有轴 `tmin` 的最大值和 `tmax` 的最小值
4. 若 `tmax >= tmin` 则光线与包围盒相交

使用 `idir`（方向的逆）避免除法运算，同时正确处理方向分量为零的情况（产生无穷大值，通过 min/max 运算自然处理）。

### 非轴对齐包围盒交叉

用于毛发曲线等使用定向包围盒（OBB）的几何体：

1. 从 BVH 节点数据获取 3x4 变换矩阵，该矩阵将世界空间变换到以 OBB 为单位立方体的局部空间
2. 将光线起点和方向变换到局部空间
3. 在局部空间中使用标准 slab 方法（等效于对单位立方体 [0,1]^3 做交叉测试）

### BVH 节点数据布局

每个 BVH2 节点占用 `bvh_nodes` 数组中连续的多个 `float4`：
- 偏移 0：`cnodes`，包含子节点可见性标志（`.x`/`.y`）和子节点地址（`.z`/`.w`）
- 偏移 1-3：轴对齐节点的包围盒数据，或非轴对齐节点的变换矩阵

## 关联文件

| 文件 | 关系 |
|---|---|
| `kernel/bvh/bvh.h` | 引入本文件 |
| `kernel/bvh/types.h` | 提供 `BVH_FEATURE` 等宏定义 |
| `kernel/bvh/traversal.h` | 遍历模板中通过 `NODE_INTERSECT` 宏调用本文件函数 |
| `kernel/bvh/local.h` | 局部遍历中同样使用 `NODE_INTERSECT` |
| `kernel/bvh/shadow_all.h` | 阴影遍历中同样使用 `NODE_INTERSECT` |
| `kernel/bvh/volume.h` | 体积遍历中同样使用 `NODE_INTERSECT` |
| `kernel/bvh/volume_all.h` | 体积遍历（多次命中）中同样使用 `NODE_INTERSECT` |
