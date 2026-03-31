# types.h - 内核核心类型、常量与数据结构总定义

## 概述

`types.h` 是 Cycles 渲染内核最核心的头文件，定义了内核运行所需的几乎全部基础类型、枚举常量、位标志和数据结构。它是整个内核代码库的类型基础，包括但不限于：内核特性标志、采样模式、光线标志、图元类型、属性系统、闭包系统、着色器数据、光源/摄像机/胶片/积分器/BVH 等核心配置结构体，以及工作块分配等运行时数据结构。文件约 1830 行，是内核中最大的单体头文件。

## 类与结构体

### 基础渲染数据结构

| 结构体 | 用途 |
|---|---|
| `differential3` | 三维微分（包含 dx, dy 的 float3） |
| `differential` | 一维微分（包含 dx, dy 的 float） |
| `RaySelfPrimitives` | 光线自相交排除信息（起始图元、对象、光源图元、光源对象） |
| `Ray` | 光线结构体：原点 P、方向 D、距离范围 tmin/tmax、时间 time、光线微分 dP/dD、自相交排除 self |
| `Intersection` | 光线交点：参数距离 t、重心坐标 u/v、图元/对象索引、图元类型 |
| `BsdfEval` | BSDF 评估结果：漫反射/光泽/总和的光谱值 |
| `LocalIntersection` | 局部交点集合（用于 SSS 等需要多个邻近交点的场景，最多 `LOCAL_MAX_HITS` 个） |

### 着色器闭包系统

| 结构体 | 用途 |
|---|---|
| `ShaderClosure` | 着色器闭包基类（16 字节对齐），包含 weight、type、sample_weight、N 以及额外数据空间 |
| `ShaderVolumeClosure` | 紧凑体积闭包存储，无法线成员，减少内存占用 |
| `ShaderVolumePhases` | 体积相函数集合（最多 `MAX_VOLUME_CLOSURE` 个） |
| `VolumeStack` | 体积栈元素：对象索引和着色器索引 |

### ShaderData（着色器数据）
渲染管线最重要的运行时结构体之一，描述表面或体积中一个着色点的完整状态：
- 几何信息: `P`(位置), `N`(平滑法线), `Ng`(几何法线), `wi`(入射方向)
- 图元信息: `type`, `shader`, `flag`, `object_flag`, `prim`, `u`, `v`, `object`
- 时间与距离: `time`, `ray_length`
- 光线微分: `dP`, `dI`, `du`, `dv`（条件编译 `__RAY_DIFFERENTIALS__`）
- 参数导数: `dPdu`, `dPdv`（条件编译 `__DPDU__`）
- 运动模糊变换: `ob_tfm_motion`, `ob_itfm_motion`（条件编译 `__OBJECT_MOTION__`）
- 闭包数组: `closure[MAX_CLOSURE]` — 固定大小闭包数组
- GPU 优化变体: `ShaderDataTinyStorage`（缩减版）、`ShaderDataCausticsStorage`（焦散专用版）

### 属性系统

| 结构体 | 用途 |
|---|---|
| `AttributeDescriptor` | 属性描述符：元素类型、数据类型、偏移量 |
| `AttributeMap` | 属性映射：全局唯一 ID（uint64）、偏移、元素类型、数据类型 |

### 内核配置结构体

| 结构体 | 用途 |
|---|---|
| `KernelCamera` | 摄像机完整参数：类型、景深、运动模糊、全景投影、立体视觉、变换矩阵、裁剪、传感器尺寸、渲染尺寸 |
| `KernelFilmConvert` | 胶片转换参数：通道偏移、缩放、曝光、阴影捕捉配置 |
| `KernelTables` | 内核查找表偏移：滤波表、GGX 能量补偿表、光泽 LTC 表、薄膜表 |
| `KernelBake` | 烘焙参数 |
| `KernelLightLinkSet` | 光链接集合 |
| `KernelData` | **内核主数据结构**，聚合所有配置（cam, bake, tables, light_link_sets）及通过 data_template.h 展开的可特化成员（background, bvh, film, integrator, svm_usage），外加设备端 BVH 句柄 |

### 场景对象与几何体

| 结构体 | 用途 |
|---|---|
| `KernelObject` | 对象：变换矩阵、体积密度、pass ID、随机数、颜色、粒子索引、属性偏移、Cryptomatte ID、光链接等 |
| `KernelCurve` | 曲线（毛发）：着色器ID、首控制点、控制点数、类型 |
| `KernelCurveSegment` | 曲线段：图元索引、类型 |

### 光源系统

| 结构体 | 用途 |
|---|---|
| `KernelSpotLight` | 聚光灯参数：方向、半径、角度、衰减等 |
| `KernelAreaLight` | 面光源参数：UV轴、尺寸、方向、面积逆等 |
| `KernelDistantLight` | 远距光源参数：角度、PDF、评估因子 |
| `KernelLight` | 光源联合体：类型 + 位置 + 着色器 + SpotLight/AreaLight/DistantLight |
| `KernelLightDistribution` | 光源分布项：总面积、图元索引、着色器标志、对象ID |
| `KernelBoundingBox` / `KernelBoundingCone` | 包围体基础几何 |
| `KernelLightTreeNode` | 光源树节点：包围盒、包围锥、能量、类型（叶/内/实例/远距）、子节点索引 |
| `KernelLightTreeEmitter` | 光源树发射体：包围锥角度、能量、三角形/光源/网格联合体 |

### 体积八叉树

| 结构体 | 用途 |
|---|---|
| `KernelOctreeRoot` | 八叉树根节点：缩放、平移、ID、着色器 |
| `KernelOctreeNode` | 八叉树节点：父节点索引、首子节点索引、密度极值 |

### 其他

| 结构体 | 用途 |
|---|---|
| `KernelParticle` | 粒子：索引、年龄、生命周期、大小、旋转、位置、速度、角速度 |
| `KernelShader` | 着色器：常量发射、Cryptomatte ID、标志、pass ID |
| `KernelWorkTile` | 工作块：坐标、尺寸、采样范围、路径索引偏移 |
| `KernelShaderEvalInput` | 着色器评估输入：对象、图元、UV 坐标 |

## 枚举与常量

### 内核特性标志（`KERNEL_FEATURE_*`）
位掩码系统，每个特性占一个 bit（共 32 位），用于编译期和运行时控制内核功能的启用/禁用：
- 着色器节点: `NODE_BSDF`, `NODE_EMISSION`, `NODE_VOLUME`, `NODE_BUMP`, `NODE_RAYTRACE`, `NODE_AOV`, `NODE_LIGHT_PATH`, `NODE_PRINCIPLED_HAIR`, `NODE_PORTAL` 等
- 渲染特性: `PATH_TRACING`, `POINTCLOUD`, `HAIR_RIBBON`, `HAIR_THICK`, `OBJECT_MOTION`, `BAKING`, `SUBSURFACE`, `VOLUME`, `TRANSPARENT`, `SHADOW_CATCHER`, `DENOISING`, `LIGHT_TREE` 等
- 特性掩码组合: `KERNEL_FEATURE_NODE_MASK_SURFACE`, `KERNEL_FEATURE_NODE_MASK_VOLUME` 等

### 采样相关枚举

- `PathTraceDimension` — 路径追踪各维度的随机数分配编号（滤波、镜头、光源、BSDF、AO、体积、SSS 等）
- `SamplingPattern` — 采样模式类型（Sobol-Burley、表格化 Sobol、蓝噪声纯/首采样/轮询、自动）

### 光线标志（`PathRayFlag`）
32 位标志，描述光线的可见性和路径状态：
- 可见性: `CAMERA`, `REFLECT`, `TRANSMIT`, `DIFFUSE`, `GLOSSY`, `SINGULAR`, `TRANSPARENT`, `VOLUME_SCATTER`, `SHADOW_OPAQUE`, `SHADOW_TRANSPARENT`
- 路径状态: `MIS_SKIP`, `DIFFUSE_ANCESTOR`, `TERMINATE_*`（多种终止条件）, `EMISSION`, `SUBSURFACE_*`, `DENOISING_FEATURES`, `SHADOW_CATCHER_*`

- `PathRayMNEE` — 流形下一事件估计(MNEE)标志

### 闭包与着色器枚举

- `ClosureLabel` — 闭包类型标签（透射、反射、漫反射、光泽、奇异、透明、体积散射等）
- `PassType` — 渲染通道类型（约 60 种：光照通道、数据通道、烘焙通道）
- `CryptomatteType` — Cryptomatte 类型（对象/材质/资产/精确）
- `FilterClosures` — 闭包过滤器（发射、漫反射、光泽、透射、透明、直接光）
- `ShaderFlag` — 着色器标志（平滑法线、投射阴影、面光源、MIS、排除类别等）
- `EmissionSampling` — 发射采样方式（无、自动、正面、背面、正反面）
- `ShaderDataFlag` — 着色器数据运行时标志（约 32 个，描述闭包特性和着色器特性）
- `ShaderDataObjectFlag` — 对象级着色器标志（holdout、运动、体积、阴影捕捉、焦散等）

### 光源与摄像机枚举

- `LightType` — 光源类型（点光、远距光、环境光、面光、聚光灯、三角形光）
- `CameraType` — 摄像机类型（透视、正交、全景、自定义）
- `PanoramaType` — 全景类型（等距柱状投影、鱼眼等距/等面积、镜球、多项式鱼眼、等角立方体面、中心圆柱）
- `MotionPosition` — 快门时间偏移（起始/中心/结束）
- `DirectLightSamplingType` — 直接光采样类型（MIS/前向/NEE）
- `LightTreeNodeType` — 光源树节点类型（实例/内部/叶子/远距）

### 几何与属性枚举

- `PrimitiveType` — 图元类型位掩码（三角形、厚曲线、带状曲线、点、体积、灯光，各有运动版本）
- `CurveShapeType` — 曲线形状（带状、厚、厚线性）
- `AttributePrimitive` — 属性图元类型（几何体/细分）
- `AttributeElement` — 属性元素级别（对象/网格/面/顶点/角/曲线/体素等）
- `AttributeStandard` — 标准属性枚举（法线/UV/切线/颜色/生成坐标/运动/粒子/体积密度/温度等，约 30 种）
- `AttributeFlag` — 属性标志

### 其他枚举

- `GuidingDistributionType` — 路径引导分布类型（视差感知VMM/方向四叉树/VMM）
- `GuidingDirectionalSamplingType` — 引导方向采样类型（乘积MIS/RIS/粗糙度）
- `KernelBVHLayout` — BVH 布局类型（BVH2/Embree/OptiX/Metal/HIPRT/EmbreeGPU 及其多设备组合）

### 核心常量宏

| 常量 | 值 | 用途 |
|---|---|---|
| `OBJECT_MOTION_PASS_SIZE` | 2 | 运动通道变换数量 |
| `FILTER_TABLE_SIZE` | 1024 | 滤波表大小 |
| `RAMP_TABLE_SIZE` | 256 | 渐变表大小 |
| `SHUTTER_TABLE_SIZE` | 256 | 快门表大小 |
| `THIN_FILM_TABLE_SIZE` | 512 | 薄膜表大小 |
| `BSSRDF_MIN_RADIUS` | 1e-8f | BSSRDF 最小半径 |
| `BSSRDF_MAX_HITS` | 4 | BSSRDF 最大交点数 |
| `BSSRDF_MAX_BOUNCES` | 256 | BSSRDF 最大弹射数 |
| `LOCAL_MAX_HITS` | 4 | 局部最大交点数 |
| `VOLUME_BOUNDS_MAX` | 1024 | 体积边界最大值 |
| `SHADER_NONE` / `OBJECT_NONE` / `PRIM_NONE` | ~0 | 空值标记 |
| `LIGHT_LINK_SET_MAX` | 64 | 最大光链接集合数 |
| `MAX_CLOSURE` | 64 | 最大闭包数量 |
| `MAX_VOLUME_STACK_SIZE` | 32 | 最大体积栈深度 |
| `MAX_VOLUME_CLOSURE` | 8 | 最大体积闭包数量 |
| `VOLUME_OCTREE_MAX_DEPTH` | 7 | 体积八叉树最大深度（分辨率上限 128） |
| `INTEGRATOR_SHADOW_ISECT_SIZE_CPU` | 1024 | CPU 阴影交点缓冲大小 |
| `INTEGRATOR_SHADOW_ISECT_SIZE_GPU` | 4 | GPU 阴影交点缓冲大小 |

## 核心函数

无函数定义（纯类型/常量定义文件）。提供两个条件编译辅助宏：
- `IF_KERNEL_FEATURE(feature)` — 编译期检查内核特性是否启用
- `IF_KERNEL_NODES_FEATURE(feature)` — 编译期检查节点特性是否启用

## 依赖关系

- **内部头文件**:
  - `kernel/device/cpu/compat.h` — CPU 端兼容性（非 GPU 时）
  - `util/projection.h` — 投影变换类型
  - `util/static_assert.h` — 静态断言对齐检查
  - `kernel/svm/types.h` — SVM 闭包类型定义（`ClosureType` 等）
  - `kernel/data_template.h` — 两次 include 用于生成配置结构体和 KernelData 成员
  - `kernel/integrator/shadow_state_template.h` / `kernel/integrator/state_template.h` — Apple Silicon 打包状态（条件编译）
  - Embree4 头文件（条件编译 `WITH_EMBREE`）
- **被引用**: 被整个内核代码库广泛引用，共计约 75 个文件，是被引用次数最多的内核头文件，包括：
  - 所有场景端代码: `scene/shader.h`, `scene/film.h`, `scene/integrator.h`, `scene/light.h`, `scene/camera.h`, `scene/attribute.h` 等
  - 所有内核子系统: 闭包(`kernel/closure/*`)、SVM(`kernel/svm/*`)、积分器(`kernel/integrator/*`)、几何(`kernel/geom/*`)、光源(`kernel/light/*`)、胶片(`kernel/film/*`)、BVH(`kernel/bvh/*`)、摄像机(`kernel/camera/*`)、OSL(`kernel/osl/*`)
  - 设备端全局变量: `kernel/device/*/globals.h`
  - 积分器主机端: `integrator/path_trace_work_gpu.cpp`, `integrator/pass_accessor.cpp` 等
  - `kernel/data_arrays.h` — 数据数组声明依赖本文件的类型

## 实现细节 / 关键算法

### 内核特性的条件编译系统

文件建立了一套两级特性控制机制：
1. **位掩码特性标志**（`KERNEL_FEATURE_*`）：运行时传递给 GPU 内核，用于自适应编译
2. **预处理器宏特性**（`__HAIR__`, `__VOLUME__`, `__SUBSURFACE__` 等）：根据 `__KERNEL_FEATURES__` 位掩码在编译期有条件地 `#undef`，实现场景级选择性编译优化

这意味着如果场景中没有体积材质，`__VOLUME__` 会被 undef，所有体积相关代码在编译期即被消除。

### 打包积分器状态（Apple Silicon 优化）

在 Apple Silicon 设备上启用 `__INTEGRATOR_GPU_PACKED_STATE__`，通过 `KERNEL_STRUCT_BEGIN_PACKED` / `KERNEL_STRUCT_MEMBER_PACKED` 宏将频繁一起访问的积分器状态字段打包到连续内存中，提高缓存命中率。

### 阴影捕捉可见性位移

`SHADOW_CATCHER_VISIBILITY_SHIFT` 和相关宏利用可见性位的高 16 位存储阴影捕捉路径的可见性信息，使同一组位域能同时编码普通路径和阴影捕捉路径的可见性需求。

### 结构体对齐

所有从 CPU 传递到设备端的结构体均要求 16 字节对齐（通过 `ccl_align(16)` 和 `static_assert_align` 确保），以保证跨设备的内存布局一致性。`float3` 被避免使用（因不同设备上大小不同），取而代之的是 `float4` 或 `packed_float3`。

## 关联文件

- `kernel/data_template.h` — 被本文件两次 include，生成内核配置结构体
- `kernel/data_arrays.h` — 内核数据数组声明，依赖本文件的类型定义
- `kernel/globals.h` — 内核全局状态入口
- `kernel/svm/types.h` — SVM 节点与闭包类型
- `kernel/svm/node_types_template.h` — SVM 节点类型列表（间接依赖）
- `kernel/integrator/state_template.h` — 积分器状态模板
- `util/projection.h` — 投影变换类型定义
- `scene/devicescene.h` — 场景端设备数据管理
