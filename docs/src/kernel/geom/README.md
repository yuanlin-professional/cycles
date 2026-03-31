# kernel/geom - 几何体求交与属性查询

## 概述

`kernel/geom` 模块实现了 Cycles 路径追踪渲染器中所有几何体类型的光线求交测试、着色数据初始化和属性插值功能。该模块支持三角形网格、曲线（毛发）、点云和体积四种基本图元类型，同时提供运动模糊支持和对象空间变换。

所有几何体数据通过 BVH（层次包围体）加速结构进行组织，求交结果通过 `ShaderData` 结构传递给下游着色管线。该模块是 CPU 和 GPU 完全兼容的纯头文件库，使用 `ccl_device` 系列宏保证跨平台编译。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `geom_intersect.h` | 工具函数 | 通用求交工具函数，包括局部求交记录管理和蓄水池采样 |
| `triangle.h` | 图元实现 | 三角形图元：法线插值、dPdu/dPdv 计算、顶点获取、属性读取 |
| `triangle_intersect.h` | 求交实现 | 三角形光线求交测试，包括局部求交和阴影求交 |
| `curve.h` | 图元实现 | 曲线（毛发）图元：属性插值、运动中心位置计算 |
| `curve_intersect.h` | 求交实现 | 曲线光线求交测试，支持粗线段和细线段模式 |
| `point.h` | 图元实现 | 点云图元：属性插值、运动中心位置计算 |
| `point_intersect.h` | 求交实现 | 点云光线求交测试（球体求交） |
| `volume.h` | 图元实现 | 体积图元：体积属性采样和插值 |
| `object.h` | 对象变换 | 对象空间与世界空间变换、BVH 实例化推入/弹出、对象元数据查询 |
| `attribute.h` | 属性系统 | 几何属性描述符查找与数据获取 |
| `shader_data.h` | 着色数据 | `ShaderData` 初始化：从光线求交、采样位置、背景、体积等来源构建着色上下文 |
| `primitive.h` | 统一接口 | 图元属性统一访问接口，根据图元类型分发到对应的属性读取函数 |
| `motion_curve.h` | 运动模糊 | 曲线运动模糊：运动关键帧插值 |
| `motion_point.h` | 运动模糊 | 点云运动模糊：运动关键帧插值 |
| `motion_triangle.h` | 运动模糊 | 三角形运动模糊：顶点运动插值 |
| `motion_triangle_intersect.h` | 运动模糊 | 运动三角形光线求交测试 |
| `motion_triangle_shader.h` | 运动模糊 | 运动三角形着色数据设置 |

## 核心类与数据结构

- **`ShaderData`**：核心着色上下文结构体，包含交点位置 `P`、法线 `N`/`Ng`、入射方向 `wi`、UV 坐标 `u`/`v`、微分 `dP`/`dI`/`du`/`dv`、图元信息及对象变换矩阵等。是几何模块与着色模块之间的核心数据桥梁。
- **`Intersection`**：光线求交结果结构体，包含参数 `t`（光线距离）、`u`/`v`（重心坐标）、图元和对象索引。
- **`LocalIntersection`**：局部求交结果，支持多次命中的蓄水池采样记录。
- **`AttributeDescriptor`**：几何属性描述符，用于定位和读取顶点颜色、UV、法线贴图等自定义属性。
- **`KernelObject`**：内核对象数据，包含变换矩阵、颜色、随机数、粒子索引、体积密度、Cryptomatte ID 等元数据。
- **`ObjectTransform` / `ObjectVectorTransform`**：对象变换矩阵类型枚举，区分正向/逆向变换以及运动向量变换。

## 内核函数入口

- **`shader_setup_from_ray()`**：从光线求交结果初始化 `ShaderData`，是最常用的着色数据设置入口。根据图元类型（三角形、曲线、点云）分发到对应的 setup 函数。
- **`shader_setup_from_sample()`**：从网格上采样位置初始化 `ShaderData`，主要用于光源采样。
- **`shader_setup_from_displace()`**：为位移着色器初始化 `ShaderData`。
- **`shader_setup_from_background()`**：为背景/环境光着色初始化 `ShaderData`。
- **`shader_setup_from_volume()`**：为体积内部着色点初始化 `ShaderData`。
- **`primitive_surface_attribute<T>()`**：统一的表面属性读取模板函数，根据图元类型分发。
- **`primitive_volume_attribute<T>()`**：统一的体积属性读取模板函数。
- **`primitive_motion_vector()`**：计算运动向量渲染通道数据。
- **`object_fetch_transform()`** / **`object_fetch_transform_motion()`**：获取对象变换矩阵（静态/运动模糊）。
- **`bvh_instance_push()` / `bvh_instance_pop()`**：BVH 实例化遍历时的光线变换推入与弹出。

## GPU 兼容性

该模块完全兼容 GPU（CUDA、OptiX、HIP、Metal、oneAPI）：

- 所有函数使用 `ccl_device`/`ccl_device_inline`/`ccl_device_forceinline` 宏标记。
- 使用 `ccl_private`/`ccl_global` 地址空间限定符区分寄存器和全局内存访问。
- 通过 `kernel_data_fetch()` 宏从全局只读内存读取几何数据。
- 运动模糊变换通过 `__OBJECT_MOTION__` 编译特性开关控制。
- 不同图元类型通过 `__HAIR__`、`__POINTCLOUD__`、`__VOLUME__` 特性宏条件编译。

## 依赖关系

### 上游依赖（本模块依赖）
- `kernel/globals.h`：全局内核数据访问
- `kernel/types.h`：基础类型定义（`Intersection`、`ShaderData`、图元类型枚举等）
- `kernel/camera/projection.h`：相机投影（用于运动向量计算）
- `kernel/sample/lcg.h`：线性同余随机数生成器（蓄水池采样）
- `kernel/util/differential.h`：微分传播工具函数

### 下游依赖（依赖本模块）
- `kernel/integrator/`：所有积分器内核均通过 `shader_setup_from_*()` 系列函数使用本模块
- `kernel/light/`：光源采样模块通过 `shader_setup_from_sample()` 设置发光体着色数据
- `kernel/svm/`：着色器虚拟机通过 `ShaderData` 读取几何信息
- `kernel/bvh/`：BVH 遍历使用本模块的求交函数和实例化变换

## 关键算法与实现细节

1. **蓄水池采样（Reservoir Sampling）**：`local_intersect_get_record_index()` 在局部求交中使用蓄水池采样算法，当命中数超过最大记录数时，以 `1/num_hits` 的概率随机替换已有记录，保证对所有命中的均匀采样。

2. **实例化变换**：对于非实例化对象（`SD_OBJECT_TRANSFORM_APPLIED`），变换已在 BVH 构建时应用到世界空间；对于实例化对象，`shader_setup_from_ray()` 会手动变换法线和微分向量到世界空间。

3. **运动模糊插值**：通过 `DecomposedTransform`（分解变换）和关键帧插值实现连续时间的对象变换。`object_fetch_transform_motion()` 根据时间参数在运动步骤间插值。

4. **背面检测**：通过 `dot(Ng, wi)` 判断，背面命中时翻转法线 `N`、几何法线 `Ng` 和微分 `dPdu`/`dPdv`，并设置 `SD_BACKFACING` 标记。

5. **光线微分传播**：使用紧凑微分格式（`differential_transfer_compact`），沿光线传播位置微分和方向微分，用于纹理抗锯齿的 mipmap 级别计算。

6. **BVH 方向钳制**：`bvh_clamp_direction()` 将接近零的方向分量钳制到极小值，避免求逆时的除零错误，使用 `8.271806E-25f` 作为最小阈值。

## 参见

- `src/kernel/bvh/` - BVH 加速结构遍历
- `src/kernel/closure/` - BSDF/BSSRDF 闭包实现
- `src/kernel/integrator/shade_surface.h` - 表面着色积分器内核
- `src/scene/geometry.cpp` - 主机端几何体数据准备
