# kernel - 渲染内核

## 概述

`kernel/` 是 Cycles 路径追踪渲染器的核心计算内核目录。该目录包含所有在 GPU 和 CPU 上执行的渲染计算代码，采用头文件（`.h`）形式组织，以便在不同的 GPU 语言（CUDA、OptiX、HIP、Metal、oneAPI）和 CPU 后端之间共享。内核系统实现了完整的路径追踪流水线，包括光线生成、场景遍历（BVH）、着色器求值（SVM/OSL）、材质闭包计算、光照采样、体积渲染以及胶片/成像平面写入等功能。

内核架构支持两种执行模式：
- **兆内核（Megakernel）模式**：将整个路径追踪循环放在单个内核中执行，适合 CPU 后端。
- **波前（Wavefront）模式**：将路径追踪拆分为多个小内核按阶段执行，适合 GPU 后端以提高并行效率。

内核代码通过条件编译宏（如 `__KERNEL_GPU__`、`__KERNEL_OPTIX__`、`__KERNEL_METAL__` 等）实现跨平台兼容性，并通过特性标志（`KERNEL_FEATURE_*`）实现基于场景的选择性编译优化。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `types.h` | 头文件 | 内核全局类型定义、常量、特性标志（`KERNEL_FEATURE_*`）、采样维度枚举等核心数据结构 |
| `globals.h` | 头文件 | `KernelGlobals` 定义的跨平台包装器，为 GPU/CPU 提供统一的全局数据访问接口 |
| `image.h` | 头文件 | 图像纹理访问的跨平台包装器，在 CPU 端引入 `device/cpu/image.h` |
| `tables.h` | 头文件 | 内核常量查找表数据（黑体辐射表、CIE 色彩匹配表、Sobol-Burley 方向向量表） |
| `data_arrays.h` | 头文件 | 通过 `KERNEL_DATA_ARRAY` 宏声明所有内核数据数组（BVH 节点、物体、三角形、曲线、灯光、着色器等） |
| `data_template.h` | 头文件 | 通过 `KERNEL_STRUCT_*` 宏定义内核结构体模板（`KernelBackground`、`KernelBVH`、`KernelFilm`、`KernelIntegrator`、`KernelSVMUsage`） |
| `CMakeLists.txt` | 构建文件 | CMake 构建配置 |

## 核心类与数据结构

### KernelGlobals
内核全局数据上下文。在 CPU 上是指向包含场景数据指针和性能分析器的结构体的指针；在 GPU 上由各设备后端自行定义（通常为空或编译期常量）。

### KernelData（data_template.h 中定义）
通过宏模板系统定义的场景参数结构体集合：
- **`KernelBackground`** - 背景/环境光参数（太阳方向、环境贴图权重、MIS 参数）
- **`KernelBVH`** - BVH 加速结构配置（根节点、几何体类型标志、曲线细分级别）
- **`KernelFilm`** - 胶片/成像平面参数（色彩空间转换矩阵、渲染通道偏移、自适应采样、Cryptomatte、降噪通道）
- **`KernelIntegrator`** - 积分器参数（弹射次数限制、光照采样配置、采样模式、体积步进、路径引导参数）
- **`KernelSVMUsage`** - 着色器虚拟机 (SVM) 节点使用情况，用于着色器特化编译

### 内核数据数组（data_arrays.h）
场景数据的 GPU 可访问数组，包括：
- BVH 节点和图元索引
- 物体变换和运动数据
- 三角形、曲线、点云几何数据
- 属性映射和属性数据
- 灯光分布和灯光树
- SVM 着色器节点指令
- 纹理信息和查找表

### 特性标志系统（types.h）
通过位掩码定义的编译期特性标志（`KERNEL_FEATURE_*`），用于根据场景需要启用/禁用功能模块：
- 着色器节点特性（BSDF、发射、体积、凹凸、光线追踪等）
- 几何体特性（毛发、点云、运动模糊）
- 渲染特性（次表面散射、体积、透明阴影、阴影捕捉器、降噪、MNEE、路径引导）

## 内核函数入口

内核根目录的文件主要定义数据结构和类型，不直接包含可执行的内核入口点。实际的内核入口函数分布在各子目录中：
- `integrator/` - 路径追踪积分器的各阶段内核（初始化、相交、着色、阴影）
- `device/` - 各 GPU/CPU 后端的设备级内核入口
- `film/` - 胶片写入和自适应采样内核
- `bake/` - 烘焙评估内核

## GPU 兼容性

内核代码通过以下机制实现跨 GPU 平台兼容：

| 平台 | 宏定义 | 加速结构 |
|------|--------|----------|
| CUDA | `__KERNEL_CUDA__` | BVH2 软件遍历 |
| OptiX | `__KERNEL_OPTIX__` | 硬件 RT Core |
| HIP (AMD) | `__KERNEL_HIP__` | BVH2 / HIPRT |
| Metal (Apple) | `__KERNEL_METAL__` | Metal RT |
| oneAPI (Intel) | `__KERNEL_ONEAPI__` | Embree GPU |
| CPU | 无 GPU 宏 | Embree 4 / BVH2 |

- `ccl_device`、`ccl_device_inline`、`ccl_global`、`ccl_private` 等跨平台修饰符在各设备兼容层中定义
- GPU 端不支持路径引导（`__PATH_GUIDING__`）、OSL 需特殊处理
- MNEE（Manifold Next Event Estimation）在 macOS < 13 的 Metal 上被禁用

## 模块架构

```
kernel/
├── types.h / globals.h / tables.h    核心类型与数据定义
├── data_arrays.h / data_template.h   场景数据数组与结构体模板
├── integrator/                       路径追踪积分器（波前内核调度）
├── device/                           设备后端（CPU, CUDA, OptiX, HIP, Metal, oneAPI）
├── closure/                          BSDF/BSSRDF/体积 闭包实现
├── bvh/                              BVH 遍历（光线-场景相交）
├── camera/                           相机光线生成
├── light/                            光源采样与灯光树
├── geom/                             几何体（三角形、曲线、点云）相交与属性
├── svm/                              着色器虚拟机节点
├── osl/                              开放着色语言 (OSL) 后端
├── film/                             胶片写入与自适应采样
├── sample/                           采样模式与随机数生成
├── bake/                             烘焙（位移、背景、曲线透明度）
└── util/                             内核工具函数
```

## 依赖关系

### 上游依赖（本模块依赖）
- `util/` - 基础数学库、类型定义（`float3`、`Transform` 等）、哈希函数、颜色工具
- `scene/` - 场景数据的 CPU 端管理（通过 `data_arrays.h` 和 `data_template.h` 共享数据布局定义）
- Embree 4 - CPU 端光线追踪加速（可选，通过 `WITH_EMBREE` 控制）
- NanoVDB - 体积数据访问（可选，通过 `WITH_NANOVDB` 控制）

### 下游依赖（依赖本模块）
- `device/` - 设备管理层，将内核加载到 GPU/CPU 执行
- `session/` - 渲染会话，调度内核执行
- `integrator/` (顶层) - 积分器管理，配置内核参数

## 参见

- `src/kernel/device/` - 各 GPU/CPU 后端设备适配层
- `src/kernel/integrator/` - 波前积分器内核实现
- `src/kernel/svm/` - 着色器虚拟机节点实现
- `src/scene/` - 场景数据管理（CPU 端）
- `src/util/` - 基础工具库
