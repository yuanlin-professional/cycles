# kernel/osl - 开放着色语言 (OSL) 集成

## 概述

本目录实现了 Cycles 渲染引擎与 **开放着色语言 (Open Shading Language, OSL)** 运行时的集成层。OSL 是由 Sony Pictures Imageworks 开发的着色语言，Cycles 在 CPU 端（以及通过 OptiX 的 GPU 端）使用它作为着色器的编译与执行后端。

本模块的核心职责包括：
- 将 Cycles 内部的着色数据 (`ShaderData`) 转换为 OSL 的 `ShaderGlobals` 结构
- 注册并管理所有 OSL 闭包 (closures) 类型（BSDF、体积、发射等）
- 提供 OSL 运行时所需的渲染服务回调（纹理查询、属性获取、矩阵变换等）
- 将 OSL 着色器执行结果（闭包树）展平为 Cycles 内核可处理的闭包列表
- 支持表面着色器、体积着色器、位移着色器和相机着色器四种着色类型
- 管理每线程的 OSL 执行上下文和全局状态

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `osl.h` | 头文件 | OSL 着色器引擎主入口；包含 `shaderdata_to_shaderglobals()` 数据转换函数、`flatten_closure_tree()` 闭包树展平算法，以及 `osl_eval_nodes<>()` 着色器执行模板函数（区分 CPU/GPU 路径） |
| `closures.cpp` | 源文件 | OSL 闭包注册与 CPU 端着色器执行实现；通过 `register_closures()` 向 OSL ShadingSystem 注册所有闭包类型；实现四种着色器类型（Surface、Volume、Displacement、Camera）的 `osl_eval_nodes<>` 模板特化 |
| `closures_setup.h` | 头文件 | 闭包设置函数集合；为每种闭包类型定义 `osl_closure_*_setup()` 函数，负责将 OSL 闭包参数转换为 Cycles 内核的 BSDF/BSSRDF/体积闭包结构并分配到 `ShaderData` 中；包含焦散过滤、层叠权重计算等逻辑 |
| `closures_template.h` | 头文件 | 闭包结构定义模板（X-Macro 模式）；通过 `OSL_CLOSURE_STRUCT_BEGIN/END/MEMBER` 宏定义所有闭包类型的参数结构，被多处 `#include` 以生成闭包参数列表、结构体定义和注册代码 |
| `compat.h` | 头文件 | OSL 版本兼容层；封装 `OSLUStringHash` 和 `OSLUStringRep` 类型别名，处理不同 OSL 库版本（1.14+ 与更早版本）之间的 API 差异 |
| `globals.h` | 头文件 | OSL 全局状态定义；包含 `OSLGlobals` 结构（ShadingSystem、TextureSystem、各类着色器状态向量）、`OSLTraceData`（trace() 调用结果）和 `OSLThreadData`（每线程执行上下文） |
| `globals.cpp` | 源文件 | `OSLThreadData` 的构造/析构/移动实现；负责创建和管理每线程的 OSL 着色上下文 (`ShadingContext`)、线程信息 (`PerThreadInfo`) 和 OIIO 线程信息 |
| `services.h` | 头文件 | `OSLRenderServices` 类声明；继承自 `OSL::RendererServices`，提供矩阵变换、属性查询、纹理采样、环境贴图、点云搜索、光线追踪 (trace) 等回调接口；定义 `OSLTextureHandle` 纹理句柄结构 |
| `services.cpp` | 源文件 | `OSLRenderServices` 的完整实现；实现所有渲染服务回调方法，包括对象/世界/相机矩阵变换、几何属性获取、纹理/环境贴图采样、IES 灯光查询、倒角/环境遮蔽特殊纹理、以及 trace()/getmessage() 光线追踪功能 |
| `services_gpu.h` | 头文件 | GPU 端渲染服务实现；提供 GPU（OptiX）环境下的 OSL 服务函数，包含坐标空间变换的 DeviceString 常量定义、属性获取、纹理采样等 GPU 专用实现 |
| `services_shared.h` | 头文件 | CPU/GPU 共享的服务函数；包含 `attribute_bump_map_normal()` 等在两种执行路径中通用的属性查询辅助函数 |
| `types.h` | 头文件 | OSL 类型系统定义；定义 `DeviceString` 跨平台字符串类型、`OSLClosureType` 枚举（所有闭包 ID）、闭包基础结构体（`OSLClosure`、`OSLClosureMul`、`OSLClosureAdd`、`OSLClosureComponent`）、`ShaderGlobals` 结构体（兼容 OSL 布局的 Cycles 版本），以及纹理句柄类型宏 |
| `camera.h` | 头文件 | OSL 相机着色器支持；提供 `cameradata_to_shaderglobals()` 传感器数据转换函数和 `osl_eval_camera()` 相机着色器执行函数（区分 CPU/GPU 路径） |

## 核心类与数据结构

### ShaderGlobals
OSL 着色器全局变量结构体，是 Cycles 对 `OSL::ShaderGlobals` 的镜像扩展。前半部分布局与 OSL 原生结构完全一致（位置 P、入射方向 I、法线 N/Ng、UV 坐标、微分信息等），后半部分添加了 Cycles 特有的字段：
- **CPU 端**：`kg`（内核全局指针）、`path_state` / `shadow_path_state`（积分器路径状态）
- **GPU 端**：`closure_pool`（闭包内存池）、`shade_index`（编码路径状态的整数）

### OSLGlobals
渲染会话级全局数据，持有：
- `ss`：OSL `ShadingSystem` 实例
- `ts`：OIIO `TextureSystem` 实例
- `surface_state` / `volume_state` / `displacement_state` / `bump_state`：按着色器索引存储的 `ShaderGroupRef` 向量
- `background_state` / `camera_state`：背景和相机着色器状态

### OSLThreadData
每线程的 OSL 执行数据，包含：
- `shader_globals`：可变的 ShaderGlobals 实例
- `tracedata`：trace() 光线追踪结果缓存
- `context`：OSL `ShadingContext`（着色执行上下文）

### OSLRenderServices
继承 `OSL::RendererServices` 的渲染服务类，实现所有 OSL 运行时回调：
- 矩阵变换（world、object、camera、NDC、raster、screen 空间）
- 属性获取（几何属性、对象属性、粒子属性、光路属性等）
- 纹理采样（2D/3D/环境贴图，支持 OIIO 和 SVM 两种纹理后端）
- 光线追踪（`trace()` 和 `getmessage()`）
- 特殊功能（倒角 `@bevel`、环境遮蔽 `@ao`、IES 灯光配置文件）

### 闭包系统
通过 X-Macro 模式 (`closures_template.h`) 统一定义所有闭包类型：
- **表面闭包**：Diffuse、OrenNayar、Translucent、DielectricBSDF、ConductorBSDF、GeneralizedSchlickBSDF、Microfacet 系列、Sheen、Toon 等
- **毛发闭包**：HairReflection、HairTransmission、ChiangHair、HuangHair
- **体积闭包**：VolumeAbsorption、VolumeHenyeyGreenstein、VolumeFournierForand、VolumeDraine、VolumeRayleigh
- **特殊闭包**：GenericEmissive、GenericBackground、Holdout、BSSRDF、RayPortalBSDF、Layer

### 着色器执行流程
`osl_eval_nodes<ShaderType>()` 为着色器执行的统一入口：
1. 调用 `shaderdata_to_shaderglobals()` 将 `ShaderData` 转换为 `ShaderGlobals`
2. 对表面着色器，先执行自动凹凸 (bump) 着色器更新法线
3. 通过 `OSL::ShadingSystem::execute()` 执行编译后的着色器组
4. 调用 `flatten_closure_tree()` 遍历 OSL 闭包树，将 Mul/Add/Layer 节点展开为加权闭包列表

## 依赖关系

### 内部依赖
- `kernel/types.h` — 内核基础类型定义
- `kernel/closure/` — BSDF、BSSRDF、体积、发射闭包的分配与求值
- `kernel/geom/` — 几何属性查询（attribute、primitive、object、curve 等）
- `kernel/util/differential.h` — 微分计算工具
- `kernel/camera/camera.h` — 相机系统
- `kernel/svm/ao.h`、`kernel/svm/bevel.h` — 环境遮蔽和倒角的 SVM 实现
- `kernel/integrator/state.h` — 积分器路径状态
- `kernel/osl/shaders/` — OSL 着色器节点源文件（编译后由 ShadingSystem 加载）

### 外部依赖
- **OpenShadingLanguage (OSL)** — `OSL::ShadingSystem`、`OSL::RendererServices`、`OSL::ShaderGlobals`、`OSL::ClosureColor` 等
- **OpenImageIO (OIIO)** — `OIIO::TextureSystem`、`OIIO::ustring`、`OIIO::unordered_map_concurrent`

## 参见

- `src/kernel/osl/shaders/` — 所有 OSL 着色器节点的源代码
- `src/kernel/svm/` — 着色器虚拟机 (SVM) 后端，与 OSL 后端并行的另一种着色器执行方式
- `src/kernel/closure/` — 闭包求值的核心实现
- `src/scene/osl.h` / `src/scene/osl.cpp` — 场景级 OSL 着色器管理（编译、链接、更新）
