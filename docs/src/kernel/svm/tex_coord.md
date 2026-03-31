# tex_coord.h - 纹理坐标与法线贴图节点的着色器虚拟机实现

## 概述

本文件实现了着色器虚拟机（SVM）中的纹理坐标（Texture Coordinate）节点、法线贴图（Normal Map）节点和切线（Tangent）节点。纹理坐标节点为纹理采样提供多种坐标空间映射；法线贴图节点将颜色纹理解码为扰动法线；切线节点输出表面切线方向。这是 Cycles 中纹理映射系统的核心组件。

## 核心函数

### `svm_node_tex_coord`

```c
ccl_device_noinline int svm_node_tex_coord(
    KernelGlobals kg,
    ccl_private ShaderData *sd,
    const uint32_t path_flag,
    ccl_private float *stack,
    const uint4 node,
    int offset)
```

**功能**：生成各种纹理坐标。

**支持的坐标类型**（`NodeTexCoord`）：

| 枚举值 | 坐标类型 | 说明 |
|--------|----------|------|
| `NODE_TEXCO_OBJECT` | 物体坐标 | 着色点位置变换到物体局部空间 |
| `NODE_TEXCO_OBJECT_WITH_TRANSFORM` | 带自定义变换的物体坐标 | 使用指令流中存储的变换矩阵 |
| `NODE_TEXCO_NORMAL` | 法线坐标 | 物体空间法线 |
| `NODE_TEXCO_CAMERA` | 相机坐标 | 着色点在相机空间中的位置 |
| `NODE_TEXCO_WINDOW` | 窗口坐标 | 着色点的 NDC 屏幕空间坐标 |
| `NODE_TEXCO_REFLECTION` | 反射坐标 | 反射向量（用于环境映射） |
| `NODE_TEXCO_DUPLI_GENERATED` | 实例生成坐标 | 实例化物体的生成坐标 |
| `NODE_TEXCO_DUPLI_UV` | 实例 UV | 实例化物体的 UV 坐标 |
| `NODE_TEXCO_VOLUME_GENERATED` | 体积生成坐标 | 体积归一化位置坐标 |

### `svm_node_tex_coord_bump_dx` / `svm_node_tex_coord_bump_dy`

```c
ccl_device_noinline int svm_node_tex_coord_bump_dx(...)
ccl_device_noinline int svm_node_tex_coord_bump_dy(...)
```

**功能**：纹理坐标节点的凹凸贴图变体。对位置相关的坐标类型添加微分偏移，用于凹凸法线计算。

**需要微分处理的类型**：物体坐标、法线坐标、相机坐标、窗口坐标、体积生成坐标。

### `svm_texco_reflection`

```c
ccl_device_inline float3 svm_texco_reflection(const ccl_private ShaderData *sd)
```

**功能**：计算反射向量。对有物体的着色点，反射入射方向关于法线的镜像。

### `svm_texco_camera`

```c
ccl_device_inline float3 svm_texco_camera(
    KernelGlobals kg,
    const ccl_private ShaderData *sd,
    const ccl_private float3 &P)
```

**功能**：将位置变换到相机空间。对无物体的着色点（如背景），先加上相机位置偏移。

### `texco_normal_from_uv`

```c
ccl_device_inline float3 texco_normal_from_uv(
    KernelGlobals kg,
    ccl_private ShaderData *sd,
    const float u, const float v)
```

**功能**：根据给定的 UV 坐标重新插值平滑法线。用于凹凸贴图变体中法线坐标的微分计算。

### `svm_node_normal_map`

```c
ccl_device_noinline void svm_node_normal_map(
    KernelGlobals kg,
    ccl_private ShaderData *sd,
    ccl_private float *stack,
    const uint4 node)
```

**功能**：将法线贴图纹理颜色解码为扰动法线。

**支持的空间模式**：

| 空间 | 说明 |
|------|------|
| 切线空间 (`NODE_NORMAL_MAP_TANGENT`) | 使用 TBN 矩阵变换，最常用 |
| 物体空间 (`NODE_NORMAL_MAP_OBJECT`) | 法线在物体局部空间中 |
| 世界空间 (`NODE_NORMAL_MAP_WORLD`) | 法线在世界空间中 |
| Blender 物体空间 (`NODE_NORMAL_MAP_BLENDER_OBJECT`) | Y/Z 分量取反（Blender 约定） |
| Blender 世界空间 (`NODE_NORMAL_MAP_BLENDER_WORLD`) | Y/Z 分量取反（Blender 约定） |

### `svm_node_tangent`

```c
ccl_device_noinline void svm_node_tangent(
    KernelGlobals kg,
    ccl_private ShaderData *sd,
    ccl_private float *stack,
    const uint4 node)
```

**功能**：生成表面切线向量。支持两种模式：
- **UV 映射**（`NODE_TANGENT_UVMAP`）：从 UV 属性获取切线
- **径向**（radial）：基于生成坐标计算绕指定轴（X/Y/Z）的径向切线

## 依赖关系

- **内部头文件**：`kernel/camera/camera.h`、`kernel/geom/motion_triangle.h`、`kernel/geom/object.h`、`kernel/geom/primitive.h`、`kernel/svm/attribute.h`、`kernel/svm/types.h`、`kernel/svm/util.h`、`util/math_base.h`
- **被引用**：`kernel/svm/svm.h`（SVM 主调度器）

## 实现细节 / 关键算法

- **切线空间法线贴图**的完整流程：
  1. 颜色值从 [0,1] 范围映射到 [-1,1]：`color = 2 * (color - 0.5)`
  2. 获取未归一化的插值法线和切线属性
  3. 对于有置换的网格，优先使用 `ATTR_STD_NORMAL_UNDISPLACED` 作为基准法线
  4. 在切线空间中应用强度：X/Y 分量乘以 strength，Z 分量使用 `mix(1, z, saturate(strength))`
  5. 使用 TBN 矩阵将颜色向量从切线空间变换到物体空间
  6. 变换到世界空间并处理背面翻转

- **Blender 坐标约定**：Blender 使用右手坐标系但 Y 轴朝上，某些法线贴图空间需要翻转 Y 和 Z 分量来匹配。

- **窗口坐标的正交相机特殊处理**：对于正交相机的直接可见光线，使用 `sd->ray_P`（光线起点）而非 `sd->P`（着色点位置）来计算 NDC 坐标，避免正交投影下的深度信息丢失。

- **径向切线计算**：通过对生成坐标取叉积来获得绕指定轴的切线方向，最后通过 `cross(N, normalize(cross(tangent, N)))` 确保切线位于表面切平面内并与法线垂直。

- **安全回退**：法线贴图在属性缺失或结果为零/非有限值时回退到 `sd->N`，保证不会产生无效法线。

## 关联文件

- `kernel/svm/svm.h`：SVM 主调度入口
- `kernel/camera/camera.h`：相机变换函数（`camera_world_to_ndc`、`camera_position`）
- `kernel/geom/object.h`：物体空间变换
- `kernel/geom/motion_triangle.h`：运动模糊三角形平滑法线
- `kernel/svm/attribute.h`：属性查找
