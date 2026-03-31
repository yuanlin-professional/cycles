# kernel/integrator - 路径追踪积分器

## 概述

`kernel/integrator` 模块实现了 Cycles 渲染器的核心路径追踪积分器。该模块支持两种执行模式：**波前路径追踪**（Wavefront Path Tracing，GPU）和**巨内核**（Megakernel，CPU），将路径追踪过程分解为多个独立的内核阶段，以优化 GPU 上的执行一致性（coherence）和占用率。

波前路径追踪的核心思想是将路径追踪循环拆分为离散的内核调度步骤。每条路径在每一步执行一个内核，然后指示下一步要执行的内核。GPU 主机端调度器按队列收集同类型的路径并批量执行，最大化 SIMD 利用率。CPU 模式下通过巨内核将所有阶段串行执行在单一函数中。

本目录包含 31 个文件，涵盖路径初始化、光线求交、表面/体积/光源着色、阴影评估、次表面散射、路径状态管理等完整的路径追踪管线。

## 目录结构

### 路径初始化

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `init_from_camera.h` | 初始化 | 从相机生成初始光线：滤波采样、运动模糊/景深采样、自适应采样收敛检查，初始化路径状态 |
| `init_from_bake.h` | 初始化 | 从烘焙任务初始化：从网格表面位置生成初始光线，用于纹理烘焙 |

### 光线求交内核

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `intersect_closest.h` | 求交 | 最近求交内核：通过 BVH 找到最近交点，同时测试解析光源，根据命中类型决定下一个着色内核 |
| `intersect_shadow.h` | 求交 | 阴影求交内核：测试阴影光线的遮挡，处理透明阴影的多次命中 |
| `intersect_subsurface.h` | 求交 | 次表面散射求交内核：在物体内部找到散射出射点 |
| `intersect_volume_stack.h` | 求交 | 体积栈求交内核：遍历场景构建当前光线所在体积的堆栈 |
| `intersect_dedicated_light.h` | 求交 | 阴影链接专用光源求交内核：对阴影链接的光源进行独立的求交测试 |

### 着色内核

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `shade_surface.h` | 着色 | 表面着色核心内核：评估表面着色器、执行直接光照（NEE）采样、BSDF 采样确定下一光线方向、写入渲染通道 |
| `shade_volume.h` | 着色 | 体积着色内核：体积散射/吸收评估、等距/随机游走体积采样、体积直接光照 |
| `shade_background.h` | 着色 | 背景着色内核：评估环境光/背景着色器、处理路径未命中任何物体的情况 |
| `shade_light.h` | 着色 | 光源着色内核：处理光线命中光源时的 MIS 权重计算和发射光谱评估 |
| `shade_shadow.h` | 着色 | 阴影着色内核：评估透明阴影衰减、累积阴影通量 |
| `shade_dedicated_light.h` | 着色 | 阴影链接专用光源着色内核 |

### 次表面散射

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `subsurface.h` | SSS 入口 | 次表面散射主入口：入射点折射、散射方法分发 |
| `subsurface_disk.h` | SSS 方法 | 盘式次表面散射（Disk SSS）：基于盘采样的 BSSRDF 求交 |
| `subsurface_random_walk.h` | SSS 方法 | 随机游走次表面散射（Random Walk SSS）：在介质内部随机游走直到出射 |

### 流形下一事件估计 (MNEE)

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `mnee.h` | 焦散 | 流形下一事件估计（Manifold Next Event Estimation）：通过折射表面的牛顿求解器找到满足费马原理的光路，用于焦散效果的高效采样 |

### 路径状态管理

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `state.h` | 状态定义 | 积分器状态数据结构定义：路径状态、阴影路径状态、体积栈、光线数据等，支持 CPU（AoS）和 GPU（SoA）布局 |
| `state_template.h` | 状态模板 | 积分器状态字段声明模板 |
| `shadow_state_template.h` | 阴影状态模板 | 阴影路径状态字段声明模板 |
| `state_flow.h` | 控制流 | 内核间控制流：`integrator_path_init/next/terminate`，管理路径在不同内核间的调度排队 |
| `state_util.h` | 状态工具 | 状态读写工具函数：光线、求交结果、体积栈的序列化/反序列化 |
| `path_state.h` | 路径状态 | 路径状态管理：反弹计数、俄罗斯轮盘赌续存概率、路径标记、RNG 状态 |

### 着色器评估

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `surface_shader.h` | 着色器 | 表面着色器评估入口：调用 SVM 执行、闭包合并、BSDF 采样和评估 |
| `volume_shader.h` | 着色器 | 体积着色器评估入口：调用 SVM 执行体积着色器、体积消光系数计算 |
| `displacement_shader.h` | 着色器 | 位移着色器评估入口：调用 SVM 执行位移计算 |

### 特殊功能

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `shadow_catcher.h` | 阴影捕获 | 阴影捕获器路径分裂逻辑：检测并处理阴影捕获器对象 |
| `shadow_linking.h` | 阴影链接 | 阴影链接调度逻辑：为阴影链接光源安排专用求交和着色 |
| `guiding.h` | 路径引导 | 路径引导（Path Guiding）集成：使用学习的辐射分布引导 BSDF 采样方向 |
| `volume_stack.h` | 体积栈 | 体积栈操作：进入/离开体积对象时更新体积栈 |
| `megakernel.h` | 巨内核 | CPU 巨内核模式：将所有内核阶段串行执行在单一循环中 |

## 核心类与数据结构

- **`IntegratorState`（CPU）/ `IntegratorStateCPU`**：路径积分器完整状态，CPU 上使用 AoS（Array of Structures）布局。包含主路径状态、阴影路径状态、AO 路径状态。
- **`IntegratorState`（GPU）**：GPU 上使用 SoA（Structure of Arrays）布局，每个字段作为独立的全局数组，通过路径索引访问。
- **路径状态字段**：`queued_kernel`（下一个内核）、`bounce`（反弹次数）、`flag`（路径标记）、`continuation_probability`（续存概率）、`throughput`（通量）、`render_pixel_index`（像素索引）等。
- **`DeviceKernel` 枚举**：所有 GPU 可调度的内核标识，用于波前排队系统。

## 波前路径追踪管线

波前路径追踪将路径追踪循环拆分为以下内核阶段，每阶段在 GPU 上作为独立的 compute kernel 批量执行：

```
初始化
  |
  v
integrator_init_from_camera  ──>  生成相机光线，初始化路径状态
  |
  v
┌─────────────────────────────────────────────────────┐
│  主路径循环                                           │
│                                                       │
│  intersect_closest  ──>  BVH 最近求交 + 光源求交       │
│       |                                               │
│       |── 命中表面 ──> shade_surface                   │
│       |                   |── NEE (直接光照采样)        │
│       |                   |── BSDF 采样 (间接光方向)    │
│       |                   └── 渲染通道写入              │
│       |                                               │
│       |── 命中体积 ──> shade_volume                    │
│       |                   |── 体积散射/吸收             │
│       |                   └── 体积直接光照              │
│       |                                               │
│       |── 命中光源 ──> shade_light                     │
│       |                   └── MIS 权重 + 发射评估       │
│       |                                               │
│       |── 未命中 ──> shade_background                  │
│       |                   └── 背景着色器评估            │
│       |                                               │
│       └── 需要光线追踪着色 ──> shade_surface_raytrace  │
│                                (AO/Bevel 节点)         │
│                                                       │
│  intersect_subsurface ──> 次表面散射求交               │
│  intersect_volume_stack ──> 体积栈构建                 │
│  intersect_dedicated_light ──> 阴影链接光源求交        │
│  shade_dedicated_light ──> 阴影链接光源着色            │
│  shade_surface_mnee ──> 流形 NEE 焦散着色              │
└─────────────────────────────────────────────────────┘
  |
  ├── 阴影路径（并行分支）
  │     intersect_shadow ──> 阴影遮挡测试
  │     shade_shadow ──> 透明阴影评估
  │
  v
路径终止 ──> 结果累积到渲染缓冲区
```

## 内核函数入口

### 主路径内核
- **`integrator_init_from_camera()`**：路径初始化，生成相机光线并初始化路径状态。
- **`integrator_intersect_closest()`**：最近求交，调用 BVH 遍历和光源求交，根据结果路由到下一个着色内核。包含俄罗斯轮盘赌路径终止逻辑。
- **`integrator_shade_surface()`**：表面着色，执行 SVM 着色器评估、NEE 直接光照采样、BSDF 采样。
- **`integrator_shade_volume()`**：体积着色，执行体积散射采样和直接光照。
- **`integrator_shade_background()`**：背景着色，评估环境光着色器。
- **`integrator_shade_light()`**：光源着色，计算 MIS 权重并累积发射光。

### 阴影路径内核
- **`integrator_intersect_shadow()`**：阴影求交，测试光源采样方向的遮挡。
- **`integrator_shade_shadow()`**：阴影着色，评估透明表面的阴影衰减。

### 控制流函数
- **`integrator_path_init()`**：初始化新路径，将其排入指定内核队列。
- **`integrator_path_next()`**：路径转移到下一个内核，更新队列计数器。
- **`integrator_path_terminate()`**：终止路径，写入最终结果。

## GPU 兼容性

该模块是 Cycles GPU 渲染的核心：

- **波前调度**：GPU 上每个内核作为独立的 compute kernel 启动。`queued_kernel` 字段和原子队列计数器实现无锁的路径-内核分配。
- **SoA vs AoS**：GPU 使用 SoA 布局最大化内存合并效率；CPU 使用 AoS 布局优化缓存局部性。通过 `INTEGRATOR_STATE()` 宏抽象差异。
- **着色器排序**：着色内核通过着色器键排序，使相同着色器的路径连续执行，提升 GPU 的指令缓存和分支一致性。
- **分离的光线追踪内核**：包含 AO/Bevel 节点的着色器使用独立的 `shade_surface_raytrace` 内核，避免常规着色内核的寄存器压力增大。
- **MNEE 专用内核**：`shade_surface_mnee` 作为独立内核处理流形 NEE，因其需要额外的求交和复杂计算。

## 依赖关系

### 上游依赖（本模块依赖）
- `kernel/bvh/bvh.h`：BVH 光线求交
- `kernel/camera/camera.h`：相机光线生成
- `kernel/film/`：渲染通道写入（light_passes、data_passes、denoising_passes、adaptive_sampling）
- `kernel/geom/`：几何体着色数据设置（`shader_setup_from_ray` 等）
- `kernel/light/`：光源采样和求交
- `kernel/svm/svm.h`：SVM 着色器评估
- `kernel/closure/`：BSDF/BSSRDF 闭包采样和评估
- `kernel/sample/`：采样模式和 RNG

### 下游依赖（依赖本模块）
- `src/device/`：设备层调度波前内核
- `src/session/`：渲染会话管理

## 关键算法与实现细节

1. **波前路径追踪调度**：每条路径维护 `queued_kernel` 字段指示下一步要执行的内核。GPU 主机端收集所有排队到同一内核的路径，批量启动对应的 compute kernel。通过原子计数器 `queue_counter->num_queued[kernel]` 跟踪每个队列的路径数。CPU 上通过 `integrator_megakernel()` 的 while 循环串行执行。

2. **俄罗斯轮盘赌（Russian Roulette）**：在 `intersect_closest` 中执行路径终止判定。续存概率基于路径通量和最大反弹次数计算。当路径被终止但当前表面有发射时，标记 `PATH_RAY_TERMINATE_ON_NEXT_SURFACE` 以便在着色内核中仍评估发射后再终止。

3. **流形下一事件估计（MNEE）**：实现了 Hanika et al. (2015) 的方法，通过伪牛顿求解器在折射表面上行走（specular manifold walk），找到满足费马原理的光路点。使用广义半向量投影的零条件作为约束，线性化后投影回折射表面。支持多层折射界面的级联求解。

4. **路径引导（Path Guiding）**：通过 `guiding.h` 集成 Intel Open Path Guiding Library (openpgl)。在路径追踪过程中学习入射辐射的空间和方向分布，使用 RIS（Resampled Importance Sampling）结合 BSDF PDF 和引导 PDF 进行混合采样。

5. **次表面散射**：支持两种方法：
   - **盘式散射**：在入射点附近的盘形区域采样出射点，使用 BSSRDF 剖面加权。
   - **随机游走**：在介质内部执行随机游走，物理上更准确。`CLOSURE_BSSRDF_RANDOM_WALK_SKIN_ID` 变体使用 50% 概率的漫反射入射采样以模拟皮肤散射。

6. **阴影捕获器路径分裂**：在阴影捕获器对象表面分裂路径为两条：一条记录阴影（无阴影捕获器的场景），一条记录遮罩物体。两条路径独立追踪并在胶片模块中合成。

## 参见

- `src/kernel/bvh/` - BVH 加速结构遍历
- `src/kernel/film/` - 渲染缓冲区和通道管理
- `src/kernel/light/` - 光源采样
- `src/kernel/svm/` - 着色器虚拟机
- `src/device/` - GPU 设备层波前调度
- `src/integrator/` - 主机端积分器管理
