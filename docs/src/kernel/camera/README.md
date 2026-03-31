# kernel/camera - 相机光线生成

## 概述

`kernel/camera/` 实现了 Cycles 渲染器的相机光线生成系统，负责将像素坐标（光栅空间）转换为世界空间中的光线。该模块支持多种相机投影类型——透视、正交、全景（等距柱形、鱼眼等距、鱼眼等立体角、镜面球、中心柱形投影）以及自定义 OSL 相机。同时实现了景深（DOF）模拟、运动模糊相机插值、立体渲染和光线微分计算。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `camera.h` | 头文件 | 相机光线生成主逻辑：透视/正交/全景/自定义相机的光线构造，景深模拟，运动模糊，立体渲染，统一入口 `camera_sample()` |
| `projection.h` | 头文件 | 相机投影数学工具：等距柱形（Equirectangular）、中心柱形（Central Cylindrical）、鱼眼等距（Fisheye Equidistant）、鱼眼等立体角（Fisheye Equisolid）、镜面球（Mirror Ball）等坐标变换 |

## 核心类与数据结构

### KernelCamera（定义于 kernel/types.h）
相机内核参数结构体，包含：
- **投影矩阵**: `rastertocamera`（光栅到相机空间）、`cameratoworld`（相机到世界空间）
- **景深参数**: `aperturesize`（光圈大小）、`focaldistance`（焦距）、`blades`（光圈叶片数）、`bladesrotation`（叶片旋转）、`inv_aperture_ratio`（变形比率，用于模拟变形镜头散景）
- **裁剪参数**: `nearclip`、`cliplength`
- **运动模糊**: `num_motion_steps`、`shuttertime`、运动插值变换
- **全景参数**: `panorama_type`、`fisheye_fov`、`fisheye_lens`、`equirectangular_range`
- **立体参数**: `interocular_offset`、`convergence_distance`、`pole_merge_angle_from/to`
- **透视运动**: `have_perspective_motion`、`perspective_pre/post`（快门前后的投影矩阵）

### Ray 结构体
光线数据，由相机模块填充：
- `P` - 光线起点（世界空间）
- `D` - 光线方向（世界空间，已归一化）
- `tmin` / `tmax` - 光线参数范围
- `time` - 运动模糊时间 [0, 1]
- `dP` / `dD` - 光线微分（紧凑存储格式）

## 内核函数入口

| 函数 | 文件 | 说明 |
|------|------|------|
| `camera_sample()` | `camera.h` | **统一相机采样入口**：根据相机类型分发到具体实现 |
| `camera_sample_perspective()` | `camera.h` | 透视相机光线生成（支持 DOF、运动模糊、立体渲染） |
| `camera_sample_orthographic()` | `camera.h` | 正交相机光线生成（支持 DOF、运动模糊） |
| `camera_sample_panorama()` | `camera.h` | 全景相机光线生成（分发到各全景投影类型） |
| `camera_sample_custom()` | `camera.h` | 自定义 OSL 相机光线生成 |
| `camera_sample_aperture()` | `camera.h` | 光圈采样：圆形盘或正多边形散景 |
| `camera_sample_to_ray()` | `camera.h` | 辅助函数：将相机空间的 P/D 变换到世界空间（含运动模糊插值和立体变换） |
| `direction_to_equirectangular()` | `projection.h` | 方向向量到等距柱形 UV 坐标 |
| `equirectangular_to_direction()` | `projection.h` | 等距柱形 UV 坐标到方向向量 |
| `fisheye_equidistant_to_direction()` | `projection.h` | 鱼眼等距投影到方向向量 |
| `fisheye_equisolid_to_direction()` | `projection.h` | 鱼眼等立体角投影到方向向量 |
| `direction_to_mirrorball()` | `projection.h` | 方向向量到镜面球 UV 坐标 |
| `direction_to_central_cylindrical()` | `projection.h` | 方向向量到中心柱形投影 UV 坐标 |

## GPU 兼容性

- 所有函数使用 `ccl_device` / `ccl_device_inline` 修饰，在 GPU 和 CPU 上均可编译
- `KernelCamera` 使用 `float4` 代替 `float3` 确保 GPU 上一致的内存对齐和填充
- 光线微分使用紧凑存储格式（`differential_make_compact`/`differential_zero_compact`）减少寄存器占用
- `ccl_constant` 修饰符用于 `KernelCamera` 参数指针，使 GPU 可利用常量内存缓存
- 自定义 OSL 相机（`camera_sample_custom`）需要 `WITH_OSL` 编译选项

## 依赖关系

### 上游依赖（本模块依赖）
- `kernel/globals.h` - `KernelGlobals`、`kernel_data.cam` 相机参数访问
- `kernel/camera/projection.h` - 投影数学函数
- `kernel/sample/mapping.h` - `sample_uniform_disk()`（光圈采样）、`regular_polygon_sample()`
- `kernel/util/differential.h` - 光线微分计算
- `kernel/util/lookup_table.h` - 查找表（快门曲线等）
- `kernel/osl/camera.h` - OSL 自定义相机支持（可选）
- `util/math.h` - 三角函数、向量运算
- `util/projection.h` - `spherical_to_direction()` 等球面坐标工具

### 下游依赖（依赖本模块）
- `kernel/integrator/` - 积分器在路径初始化阶段调用 `camera_sample()` 生成初始光线
- `kernel/bake/bake.h` - 烘焙模块使用 `projection.h` 中的等距柱形投影
- `kernel/light/` - 背景光采样使用投影变换

## 关键算法与实现细节

### 透视相机与景深（camera.h）
1. 将光栅坐标通过 `rastertocamera` 变换到相机空间
2. 如果有透视运动模糊，在快门前后的投影矩阵之间插值
3. 景深模拟：在光圈平面上采样一个点，计算穿过焦平面上对应焦点的光线方向
4. 光圈形状支持圆形盘（`blades == 0`）或正多边形散景（`blades > 0`），并可通过 `inv_aperture_ratio` 模拟变形镜头的椭圆散景

### 立体渲染（camera.h）
使用球面立体变换（`spherical_stereo_transform`）实现双目立体渲染：
- 根据 `interocular_offset` 偏移相机位置
- 在极点附近通过 `pole_merge_angle_from/to` 参数平滑合并左右眼视图，避免极点处的伪影
- 立体模式下光线微分需要从头计算（使用相邻像素差分），而非简单的参数传递

### 全景投影类型（projection.h）
支持多种全景投影的正向/反向变换：
- **等距柱形**: 经纬度映射，支持自定义角度范围
- **鱼眼等距**: 角度与距离成线性关系
- **鱼眼等立体角**: 角度与投影距离的正弦成线性关系
- **镜面球**: 模拟球面反射镜成像
- **中心柱形**: 柱面投影变体

## 参见

- `src/kernel/integrator/init_from_camera.h` - 相机光线采样的调用点
- `src/kernel/sample/mapping.h` - 圆盘和多边形采样工具
- `src/kernel/util/differential.h` - 光线微分数学
- `src/kernel/osl/camera.h` - OSL 自定义相机实现
