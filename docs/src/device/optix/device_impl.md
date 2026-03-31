# device_impl.h / device_impl.cpp - OptiX 设备核心实现

## 概述

本文件是 OptiX 渲染设备的核心实现，定义了 `OptiXDevice` 类及其完整的生命周期管理。该类继承自 `CUDADevice`，在 CUDA 基础上扩展了 OptiX 光线追踪管线的创建、内核加载、层次包围体(BVH)构建与管理等功能。文件包含约 1900 行代码，是整个 OptiX 后端中最大且最关键的实现文件。

## 类与结构体

### `OptiXDevice`
- **继承**: `CUDADevice`（公有继承）
- **功能**: 封装 NVIDIA OptiX 光线追踪 API，管理 OptiX 上下文、模块、管线、程序组和着色器绑定表(SBT)的完整生命周期，提供路径追踪所需的光线求交与着色管线，以及层次包围体(BVH)加速结构的构建与更新。
- **关键成员**:
  - `context` (`OptixDeviceContext`) — OptiX 设备上下文句柄
  - `optix_module` (`OptixModule`) — 包含所有 OptiX 内核的主模块
  - `builtin_modules[4]` (`OptixModule[]`) — 内置求交模块（用于粗曲线和线性曲线，各含运动模糊和非运动模糊变体）
  - `pipelines[NUM_PIPELINES]` (`OptixPipeline[]`) — OptiX 管线数组，包含 `PIP_SHADE`（着色管线）和 `PIP_INTERSECT`（求交管线）
  - `groups[NUM_PROGRAM_GROUPS]` (`OptixProgramGroup[]`) — 所有程序组（光线生成、未命中、命中、可调用）
  - `pipeline_options` (`OptixPipelineCompileOptions`) — 管线编译选项，含运动模糊、图元类型标志等配置
  - `sbt_data` (`device_vector<SbtRecord>`) — 着色器绑定表数据，存储在设备内存中
  - `launch_params` (`device_only_memory<KernelParamsOptiX>`) — 内核启动参数缓冲区
  - `tlas_handle` (`OptixTraversableHandle`, 私有) — 顶层加速结构(TLAS)的可遍历句柄
  - `delayed_free_bvh_memory` (私有) — 延迟释放的 BVH 内存队列，防止 GPU 渲染时释放几何数据
  - `osl_globals` (`OSLGlobals`, 条件编译) — OSL 全局状态
  - `osl_modules` / `osl_groups` (条件编译) — OSL 着色器模块和程序组
- **关键方法**:
  - `OptiXDevice()` — 构造函数，初始化 CUDA 上下文后创建 OptiX 设备上下文，配置日志回调及调试模式，分配启动参数缓冲区
  - `~OptiXDevice()` — 析构函数，按序销毁所有 OptiX 资源（管线、程序组、模块、上下文），释放设备内存
  - `load_kernels()` — 加载并编译 OptiX 内核，创建所有程序组和管线
  - `load_osl_kernels()` — 加载 OSL 着色器内核，创建 OSL 专用管线
  - `build_bvh()` — 为场景几何体构建 OptiX 加速结构
  - `build_optix_bvh()` — 底层 BVH 构建实现
  - `release_bvh()` — 延迟释放 BVH 内存
  - `const_copy_to()` — 将常量数据复制到设备，同时更新启动参数
  - `gpu_queue_create()` — 创建 `OptiXDeviceQueue` 实例
  - `get_bvh_layout_mask()` — 返回 `BVH_LAYOUT_OPTIX`
  - `compile_kernel_get_common_cflags()` — 获取含 OptiX SDK 路径的编译标志
  - `create_optix_module()` — 使用任务并行方式异步创建 OptiX 模块
  - `update_launch_params()` — 将数据写入设备端启动参数缓冲区的指定偏移处

### `SbtRecord`
- **功能**: 着色器绑定表(Shader Binding Table)的单条记录
- **关键成员**: `header[OPTIX_SBT_RECORD_HEADER_SIZE]` — OptiX SBT 记录头部数据

### 程序组枚举（匿名 `enum`）
- **功能**: 定义所有 OptiX 程序组的索引，共 `NUM_PROGRAM_GROUPS` 个，包括：
  - **光线生成组** (`PG_RGEN_*`): 19 个，涵盖求交（closest/shadow/subsurface/volume_stack/dedicated_light）、着色（background/light/surface/volume/shadow 等）、求值（displace/background/curve_shadow_transparency/volume_density）以及相机初始化
  - **未命中组** (`PG_MISS`): 1 个
  - **命中组** (`PG_HIT*`): 24 个，按几何类型（三角形默认、曲线线性/带状、点云）和功能（默认命中 D、阴影全记录 S、局部 L、体积 V）以及运动模糊变体组合
  - **可调用组** (`PG_CALL_SVM_AO`、`PG_CALL_SVM_BEVEL`): 2 个，用于着色器光线追踪中的 AO 和 Bevel 节点

### 管线枚举
- `PIP_SHADE` — 着色管线（OSL/着色器光线追踪共用）
- `PIP_INTERSECT` — 纯求交管线

## 核心函数

### `execute_optix_task()` (静态)
- **功能**: 递归执行 OptiX 异步编译任务。调用 `optixTaskExecute()` 后将产生的子任务推入 `TaskPool` 并行执行，实现多线程模块编译。

### `get_optix_include_dir()` (静态)
- **功能**: 获取 OptiX SDK 头文件目录。优先读取 `OPTIX_ROOT_DIR` 环境变量，其次使用编译时定义的 `CYCLES_RUNTIME_OPTIX_ROOT_DIR`。

### `OptiXDevice::load_kernels()`
- **功能**: 完整的内核加载流程：
  1. 检测 OSL 着色/相机功能标志
  2. 验证 PTX 文件或 OptiX SDK 是否可用
  3. 先加载 CUDA 基础内核（`CUDADevice::load_kernels()`）
  4. 销毁已有 OptiX 模块和管线
  5. 配置模块编译选项（优化级别、调试级别）和管线选项（运动模糊、图元类型、payload 数量等）
  6. 加载并编译 PTX 为 OptiX 模块（支持任务并行）
  7. 创建全部程序组：光线生成、未命中、命中（含内置曲线求交模块）、可调用
  8. 创建着色器绑定表(SBT)并上传到设备
  9. 创建着色管线和求交管线，计算并设置栈大小

### `OptiXDevice::load_osl_kernels()`
- **功能**: 加载 OSL 着色器内核。遍历所有 OSL 着色器组（相机、表面、体积、位移、凹凸），提取 PTX 代码，为每个创建 OptiX 模块和可调用程序组。还加载 `osl_services` 和 `shadeops` 辅助模块，更新 SBT，创建 OSL 专用着色管线，并将 OSL 颜色系统数据转换为 GPU 格式后上传。

### `OptiXDevice::build_bvh()`
- **功能**: 根据几何体类型构建 OptiX 加速结构：
  - **曲线(Hair)**: 根据曲线形状选择内置求交（粗圆/线性圆）或自定义 AABB 求交（带状），支持运动模糊多帧关键点
  - **网格(Mesh)/体积(Volume)**: 构建三角形 BVH，使用 `OPTIX_BUILD_INPUT_TYPE_TRIANGLES`
  - **点云(PointCloud)**: 构建自定义 AABB 原语 BVH
  - **顶层(TLAS)**: 将所有对象实例组装为实例数组，处理 SBT 偏移（不同几何类型使用不同命中程序组）、可见性掩码和运动变换(SRT Motion Transform)

### `OptiXDevice::build_optix_bvh()`
- **功能**: 底层 BVH 构建实现。使用静态互斥锁串行化构建（防止并行构建耗尽显存）。根据 BVH 类型选择快速追踪（`PREFER_FAST_TRACE` + 压缩）或快速更新（`PREFER_FAST_BUILD` + 允许更新）模式。构建完成后尝试压缩加速结构以节省显存。

### `OptiXDevice::const_copy_to()`
- **功能**: 重写基类方法，在将常量数据写入 CUDA 模块的同时，也更新 OptiX 启动参数缓冲区。对 `"data"` 常量特殊处理：注入当前设备的 TLAS 句柄到 `KernelData::device_bvh` 字段。

## 依赖关系

- **内部头文件**:
  - `device/optix/device_impl.h` — 自身头文件
  - `device/optix/queue.h` — OptiX 命令队列
  - `device/optix/util.h` — OptiX 错误检查宏
  - `device/cuda/device_impl.h` — CUDA 设备基类实现
  - `bvh/bvh.h`、`bvh/optix.h` — BVH 数据结构
  - `scene/hair.h`、`scene/mesh.h`、`scene/object.h`、`scene/pointcloud.h`、`scene/scene.h` — 场景几何体
  - `kernel/osl/globals.h` — OSL 全局状态
  - `kernel/device/optix/globals.h` — `KernelParamsOptiX` 定义
  - `util/debug.h`、`util/log.h`、`util/path.h`、`util/progress.h`、`util/task.h`
- **被引用**:
  - `src/device/optix/device.cpp` — 工厂函数中创建 `OptiXDevice` 实例
  - `src/device/optix/queue.cpp` — 命令队列访问 `OptiXDevice` 成员
  - `src/integrator/denoiser_optix.cpp` — OptiX 降噪器访问设备上下文

## 实现细节 / 关键算法

1. **任务并行模块编译**: 使用 `optixModuleCreateWithTasks()` + `TaskPool` 实现多线程 PTX 编译，`execute_optix_task()` 递归分发子任务，显著加速大型内核的编译。

2. **管线栈大小计算**: 遍历所有程序组的栈大小（`cssRG`、`cssCH`、`cssAH`、`cssIS`、`dssDC`），取各类程序的最大值，按公式 `css = max(cssRG) + maxTraceDepth * max(trace_css)` 计算，确保管线执行时不溢出。

3. **BVH 构建与压缩**: 先构建未压缩的加速结构，通过 `OPTIX_PROPERTY_TYPE_COMPACTED_SIZE` 查询压缩后大小，若有节省则调用 `optixAccelCompact()` 生成压缩版本并交换内存指针。

4. **延迟 BVH 释放**: `release_bvh()` 不立即释放 BVH 内存，而是将其移入 `delayed_free_bvh_memory` 队列，在下次 `build_bvh()` 开始时通过 `free_bvh_memory_delayed()` 统一释放。这防止了 GPU 仍在渲染时释放加速结构内存导致的崩溃。

5. **SBT 偏移机制**: 不同几何类型（三角形、粗曲线、带状曲线、点云）通过实例的 `sbtOffset` 字段选择不同的命中程序组，偏移值为 `PG_HITX_* - PG_HITD`。

6. **运动模糊支持**: 在管线选项中启用 `usesMotionBlur` 后，切换遍历图标志为 `ALLOW_ANY`（不再限于两级实例化），为运动对象创建 SRT 运动变换并通过 `optixConvertPointerToTraversableHandle()` 获取可遍历句柄。

7. **OSL 集成**: OSL 着色器通过 OptiX 的可调用程序（direct callable）机制执行。每个 OSL 着色器组编译为独立的 OptiX 模块，注册为可调用程序组，追加到 SBT 末尾。空材质默认回退到 `__direct_callable__dummy_services` 以避免崩溃。

## 关联文件

- `src/device/optix/device.h` / `device.cpp` — 设备注册与工厂入口
- `src/device/optix/queue.h` / `queue.cpp` — 命令队列实现
- `src/device/optix/util.h` — 错误检查工具宏
- `src/device/cuda/device_impl.h` / `device_impl.cpp` — CUDA 设备基类
- `src/bvh/optix.h` — `BVHOptiX` 数据结构定义
- `src/kernel/device/optix/globals.h` — `KernelParamsOptiX` 启动参数结构
- `src/integrator/denoiser_optix.cpp` — OptiX 降噪器
