# projection.h - 投影变换与坐标系转换

## 概述

`projection.h` 定义了 Cycles 渲染器中的投影变换系统，包括 `ProjectionTransform` 4x4 投影矩阵结构体、透视/正交投影矩阵构建函数，以及球面坐标、极坐标与笛卡尔坐标之间的转换工具。该文件是相机系统和坐标变换的核心组件。

## 核心函数

### 坐标系转换
| 函数 | 说明 |
|------|------|
| `direction_to_spherical(dir)` | 方向向量 → 球面坐标 (theta, phi) |
| `spherical_to_direction(theta, phi)` | 球面坐标 → 单位方向向量 |
| `spherical_cos_to_direction(cos_theta, phi)` | 余弦形式球面坐标 → 方向向量 |
| `polar_to_cartesian(r, phi)` | 极坐标 → 2D 笛卡尔坐标 |
| `to_global(p, X, Y)` / `to_global(p, X, Y, Z)` | 局部坐标 → 全局坐标 |
| `to_local(p, X, Y)` / `to_local(p, X, Y, Z)` | 全局坐标 → 局部坐标 |
| `disk_to_hemisphere(float2)` | 2D 圆盘上的点 → 半球面上的点 |

### ProjectionTransform 结构体
- 4x4 矩阵，以四个 `float4` 行向量 (x, y, z, w) 存储
- 可从 `Transform`（3x4 仿射变换）构造
- 支持 `PerspectiveMotionTransform`（运动模糊的前后两帧投影）

### 投影变换操作
| 函数 | 说明 |
|------|------|
| `transform_perspective(t, a)` | 透视投影变换（含齐次除法 w） |
| `transform_perspective_direction(t, a)` | 方向向量的透视变换（无平移） |
| `transform_perspective_deriv(t, a, dx, dy, &out_dx, &out_dy)` | 带导数的透视变换（用于纹理微分） |
| `make_projection(a..p)` | 从 16 个标量构造投影矩阵 |
| `projection_identity()` | 单位投影矩阵 |
| `projection_transpose(a)` | 矩阵转置 |
| `projection_inverse(a)` | 矩阵求逆（调用 `projection_inverse_impl`） |
| `operator*(a, b)` | 矩阵乘法 |
| `projection_to_transform(a)` | 提取 3x4 仿射部分 |

### 相机投影构建（仅 CPU）
| 函数 | 说明 |
|------|------|
| `projection_perspective(fov, near, far)` | 透视投影矩阵 |
| `projection_orthographic(znear, zfar)` | 正交投影矩阵 |

## 依赖关系

- **内部头文件**: `util/transform.h`（Transform 结构体和 transform_scale 等函数）
- **间接包含**: `util/projection_inverse.h`（非 Metal 平台）
- **被引用**: `util/transform.cpp`, `scene/camera.h`, `kernel/types.h`, `kernel/sample/mapping.h`, `kernel/closure/volume_util.h`, `app/cycles_xml.cpp`

## 实现细节 / 关键算法

1. **透视投影变换**: `transform_perspective` 将 3D 点转换为齐次坐标 `(x, y, z, 1)`，分别与矩阵四行做点积，最后除以 w 分量。w=0 时返回零向量避免除零。

2. **带导数的透视变换**: `transform_perspective_deriv` 应用商法则计算透视除法的导数：`d(c/w)/dx = (dc/dx - (dw/dx)*c) / w`。这在光线微分和纹理 LOD 计算中至关重要。

3. **矩阵乘法**: 通过先转置右矩阵，再用行-行点积实现行*列乘法。支持 `ProjectionTransform * ProjectionTransform`、`ProjectionTransform * Transform` 和 `Transform * ProjectionTransform` 三种组合。

4. **透视投影矩阵**: 标准的 OpenGL 风格透视矩阵 `f/(f-n)` 和 `-fn/(f-n)` 作为深度映射，再通过 `1/tan(fov/2)` 的缩放矩阵调整视场角。

## 关联文件

- `util/transform.h` — 3x4 仿射变换 Transform 类型
- `util/projection_inverse.h` — 4x4 矩阵求逆的 Gauss-Jordan 实现
- `scene/camera.h` — 相机系统使用 ProjectionTransform 存储投影矩阵
