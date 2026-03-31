# device_impl.h / device_impl.cpp - HIPRT 设备实现：基于 AMD HIPRT 硬件光线追踪的 GPU 设备后端

## 概述

本文件实现了 Cycles 渲染器中基于 AMD HIPRT（HIP Ray Tracing）硬件加速光线追踪的 GPU 设备后端。`HIPRTDevice` 类继承自 `HIPDevice`，在 HIP 通用 GPU 计算基础上增加了 HIPRT 硬件光线追踪能力，负责管理 HIPRT 上下文的创建与销毁、层次包围体（BVH）加速结构的构建（包含 BLAS 底层加速结构和 TLAS 顶层加速结构），以及内核编译与加载。该实现支持三角形网格、曲线毛发、点云等多种几何体类型，并完整支持运动模糊（Motion Blur）。

## 类与结构体

### HIPRTDevice

- **继承**: `HIPDevice`（位于 `device/hip/device_impl.h`）
- **功能**: 作为 HIPRT 硬件光线追踪设备的核心实现，管理 HIPRT 上下文生命周期、加速结构构建、内核编译加载、以及主机到 GPU 的场景数据传输。
- **关键成员**:
  - `hiprt_context` (`hiprtContext`) — HIPRT 上下文句柄，所有 HIPRT API 调用的基础
  - `scene` (`hiprtScene`) — HIPRT 场景对象，代表顶层加速结构（TLAS）
  - `functions_table` (`hiprtFuncTable`) — HIPRT 函数表，存储自定义相交和过滤函数，按图元类型和过滤器类型索引
  - `global_stack_buffer` (`hiprtGlobalStackBuffer`) — HIPRT 全局遍历栈缓冲区，用于 GPU 端光线遍历
  - `scratch_buffer` (`device_vector<char>`) — 构建加速结构时使用的临时缓冲区
  - `scratch_buffer_size` (`size_t`) — 当前临时缓冲区的大小
  - `hiprt_mutex` (`thread_mutex`) — 线程互斥锁，保护加速结构构建过程中的临界区
  - `stale_bvh` (`vector<hiprtGeometry>`) — 延迟释放的 BLAS 几何体列表，防止 GPU 访问已释放内存
  - `use_motion_blur` (`bool`) — 当前场景是否启用运动模糊
  - `prim_visibility` (`device_vector<uint32_t>`) — 各实例的可见性掩码，传递给 HIPRT 并用于自定义图元
  - `instance_transform_matrix` (`device_vector<hiprtFrameMatrix>`) — 实例变换矩阵，从 Cycles 的 `Transform` 格式转换为 HIPRT 的 `hiprtFrameMatrix` 格式
  - `transform_headers` (`device_vector<hiprtTransformHeader>`) — 变换头信息，映射实例 ID 到变换矩阵索引，包含帧数量和帧索引
  - `user_instance_id` (`device_vector<int>`) — 用户实例 ID 映射表，用于将 HIPRT 返回的实例 ID 映射回 Blender 原始实例 ID
  - `hiprt_blas_ptr` (`device_vector<hiprtInstance>`) — 有效的 BLAS 指针列表，传递给 HIPRT 构建 TLAS
  - `blas_ptr` (`device_vector<uint64_t>`) — 所有几何体的 BLAS 指针（含空指针），可通过几何体 ID 直接索引，用于次表面散射等
  - `custom_prim_info` (`device_vector<int2>`) — 自定义图元信息，`.x` 为图元索引，`.y` 为图元类型
  - `custom_prim_info_offset` (`device_vector<int2>`) — 每个实例在 `custom_prim_info` 中的偏移量
  - `prims_time` (`device_vector<float2>`) — 运动模糊图元的时间区间
  - `prim_time_offset` (`device_vector<int>`) — 图元时间偏移量

- **关键方法**:
  - `HIPRTDevice(const DeviceInfo &info, Stats &stats, Profiler &profiler, bool headless)` — 构造函数，初始化 HIPRT 上下文、函数表、设备向量，并设置日志级别
  - `~HIPRTDevice()` — 析构函数，释放所有设备内存、销毁全局栈缓冲区、函数表、场景和上下文
  - `get_bvh_layout_mask()` — 返回 `BVH_LAYOUT_HIPRT`，标识使用 HIPRT 加速结构布局
  - `gpu_queue_create()` — 创建并返回 `HIPRTDeviceQueue` 实例
  - `compile_kernel_get_common_cflags()` — 在父类编译标志基础上追加 `-D __HIPRT__` 宏定义
  - `compile_kernel()` — 编译 HIPRT 内核：优先查找预编译 fatbin（`.hipfb.zst`），其次查找缓存，最后调用 hipcc 编译器进行实时编译
  - `load_kernels()` — 加载内核模块，记录运动模糊状态，加载 fatbin 到 HIP 模块，并执行测试内核验证正确性
  - `const_copy_to()` — 将主机端常量数据复制到 GPU 端 `KernelParamsHIPRT` 结构体，并在复制 `KernelData` 时注入 HIPRT 场景指针
  - `build_bvh()` — BVH 构建入口：对底层几何体调用 `build_blas()`，对顶层场景调用 `build_tlas()`
  - `release_bvh()` — 将待释放的 BLAS 几何体加入延迟释放列表
  - `prepare_triangle_blas()` — 准备三角形网格的 BLAS 构建输入，支持静态三角形和运动模糊三角形（作为自定义 AABB 图元）
  - `prepare_curve_blas()` — 准备曲线毛发的 BLAS 构建输入，所有曲线段作为自定义 AABB 图元处理
  - `prepare_point_blas()` — 准备点云的 BLAS 构建输入，所有点作为自定义 AABB 图元处理
  - `build_blas()` — 构建底层加速结构（BLAS），根据几何体类型分发到对应的准备函数，管理临时缓冲区，调用 `hiprtBuildGeometry()`
  - `build_tlas()` — 构建顶层加速结构（TLAS），收集所有有效实例的变换矩阵、可见性掩码、BLAS 指针，设置函数表，调用 `hiprtBuildScene()`
  - `free_bvh_memory_delayed()` — 延迟释放 `stale_bvh` 中的 BLAS 几何体内存
  - `get_hiprt_context()` — 返回 HIPRT 上下文句柄

### 枚举类型

#### Filter_Function（protected）
- `Closest = 0` — 最近相交过滤
- `Shadows` — 阴影相交过滤
- `Local` — 局部相交过滤（用于次表面散射等）
- `Volume` — 体积相交过滤
- `Max_Intersect_Filter_Function` — 过滤函数类型总数

#### Primitive_Type（protected）
- `Triangle = 0` — 三角形图元
- `Curve` — 曲线图元
- `Motion_Triangle` — 运动模糊三角形图元
- `Point` — 点云图元
- `Max_Primitive_Type` — 图元类型总数

### 前向声明的类
- `Mesh` — 三角形网格几何体
- `Hair` — 毛发曲线几何体
- `PointCloud` — 点云几何体
- `Geometry` — 几何体基类
- `Object` — 场景对象
- `BVHHIPRT` — HIPRT 专用 BVH 数据结构

## 核心函数

### `get_hiprt_transform(float matrix[][4], Transform &tfm)`（静态辅助函数）
将 Cycles 的 `Transform`（3x4 矩阵，行主序）转换为 HIPRT 的 `float[3][4]` 矩阵格式。仅复制前三行（旋转+平移），第四行隐含为 `[0, 0, 0, 1]`。

### `compile_kernel()`
内核编译流程：
1. 获取 GPU 架构标识（`hipDeviceArch`）
2. 非自适应编译模式下，优先检查预编译 fatbin（`lib/<name>_rt_<arch>.hipfb.zst`）
3. 计算源文件 MD5 和编译标志 MD5，拼接为缓存键
4. 检查本地缓存的 fatbin（`cycles_<name>_<arch>_<md5>.hipfb`）
5. Windows 上若无预编译内核且不支持自适应编译，报错返回
6. 定位 hipcc 编译器，检查 HIP 版本（要求 >= 4.0）
7. 使用 hipcc 编译，附加 HIPRT 头文件路径 `kernel/device/hiprt` 作为额外 include 目录
8. 编译选项包含 `-ffast-math -O3 -std=c++17`

### `load_kernels()`
内核加载流程：
1. 若 HIP 模块已加载则跳过
2. 记录运动模糊特性标志
3. 调用 `compile_kernel()` 获取 fatbin 路径
4. 加载压缩的 fatbin 数据到 HIP 模块
5. 加载所有设备内核函数指针
6. 创建临时 `HIPRTDeviceQueue` 执行测试内核（选择最复杂的表面着色内核），验证内核可正常执行

### `build_bvh()`
BVH 构建入口：
1. 释放之前延迟释放的 BVH 内存
2. 设置构建选项为高质量构建（`hiprtBuildFlagBitPreferHighQualityBuild`）
3. 若为底层 BVH（单个几何体），调用 `build_blas()`
4. 若为顶层 BVH（场景），销毁旧场景后调用 `build_tlas()`

### `build_tlas()`
TLAS 构建流程：
1. 遍历所有场景对象，收集可见性、变换矩阵、BLAS 指针
2. 处理实例 ID 映射（`user_instance_id`），因为无有效几何体的对象会被 HIPRT 跳过
3. 汇总自定义图元信息和时间偏移到场景级别
4. 处理运动模糊实例的多帧变换矩阵
5. 将函数表指针写入 GPU 端 `KernelParamsHIPRT` 的四个遍历表槽位
6. 将所有缓冲区复制到 GPU
7. 调用 `hiprtCreateScene()` 和 `hiprtBuildScene()` 构建场景加速结构
8. 复制自定义图元信息和时间数据到设备端
9. 构建完成后释放临时缓冲区

## 依赖关系

### 内部头文件
- `device/hip/device_impl.h` — 父类 `HIPDevice` 定义
- `device/hip/kernel.h` — HIP 内核管理
- `device/hip/queue.h` — HIP 设备队列基类
- `device/hip/util.h` — HIP 工具函数
- `device/hiprt/queue.h` — HIPRT 设备队列
- `kernel/device/hiprt/globals.h` — HIPRT 内核参数结构体 `KernelParamsHIPRT`
- `bvh/hiprt.h` — `BVHHIPRT` 数据结构定义
- `scene/hair.h` — 毛发几何体定义
- `scene/mesh.h` — 网格几何体定义
- `scene/object.h` — 场景对象定义
- `scene/pointcloud.h` — 点云几何体定义
- `util/log.h`、`util/md5.h`、`util/path.h`、`util/progress.h`、`util/string.h`、`util/time.h`、`util/types.h` — 各类工具库

### 外部依赖
- `hiprtew.h`（动态加载模式）或 `hiprt/hiprt_types.h`（静态链接模式） — HIPRT API 类型定义

### 被引用
- `src/device/hiprt/device_impl.cpp` — 本头文件的实现文件
- `src/device/hiprt/queue.cpp` — HIPRT 队列实现中引用设备类
- `src/device/hip/device.cpp` — HIP 设备工厂中创建 HIPRT 设备实例
- `src/bvh/hiprt.cpp` — HIPRT BVH 实现中引用设备接口
- `src/device/CMakeLists.txt` — 构建系统配置

## 实现细节 / 关键算法

### 延迟 BVH 内存释放机制
HIPRT 设备采用延迟释放策略管理 BLAS 内存。当 BVH 析构时（`release_bvh()`），BLAS 几何体指针被加入 `stale_bvh` 列表而非立即销毁。这是因为即使在同步之后，GPU 作业仍可能在同步与释放之间启动，访问已释放的内存会导致崩溃。实际释放发生在下次 `build_bvh()` 调用时的 `free_bvh_memory_delayed()` 中，此时可以确保没有 GPU 作业正在使用这些资源。

### 运动模糊处理
运动模糊的实现分为两个层级：
1. **图元级运动模糊**：对于三角形、曲线和点云，当存在运动数据时，将图元转化为自定义 AABB 图元。时间区间被细分为多个 BVH 步（由 `num_motion_*_steps` 参数控制），每个时间子段计算包围盒并生成独立的 AABB 条目。每个条目记录时间范围（`prims_time`）以便在相交测试时插值。
2. **实例级运动模糊**：对于对象变换动画，多帧变换矩阵通过 `hiprtTransformHeader` 的 `frameCount` 和 `frameIndex` 关联到 `instance_transform_matrix` 数组中的连续帧。

### 实例 ID 映射
HIPRT 内部按 BLAS 指针传入顺序分配实例 ID，但无有效几何体的对象（如平面）会被跳过，导致 HIPRT 返回的实例 ID 与应用层实例 ID 不一致。`user_instance_id` 向量提供从 HIPRT 实例 ID 到原始 Blender 实例 ID 的映射。`blas_ptr` 向量则包含所有实例（含无效的）的 BLAS 指针，可通过原始 ID 直接索引，用于次表面散射等需要直接查询几何体的场景。

### 自定义图元体系
曲线、点云以及运动模糊三角形均作为自定义 AABB 图元（`hiprtPrimitiveTypeAABBList`）处理，而非 HIPRT 原生三角形。这些自定义图元需要在 GPU 端执行自定义相交测试。`custom_prim_info` 存储图元索引和类型，`custom_prim_info_offset` 提供每个实例的偏移，使 GPU 着色器能够根据 HIPRT 返回的局部图元 ID 还原出全局图元信息。

### 临时缓冲区管理
BLAS 和 TLAS 构建共享同一个 `scratch_buffer`。构建过程中，先查询所需临时缓冲区大小，若大于当前分配量则重新分配。BLAS 构建在互斥锁保护下进行以避免并发冲突。TLAS 构建完成后释放临时缓冲区以节省内存。

### 函数表设置
HIPRT 函数表按 `[图元类型][过滤函数类型]` 二维索引组织，大小为 `Max_Primitive_Type x Max_Intersect_Filter_Function`。在 TLAS 构建过程中，函数表指针被写入 GPU 端 `KernelParamsHIPRT` 的四个遍历表槽位（最近相交、阴影、局部、体积），供内核遍历时调用对应的自定义相交/过滤函数。

## 关联文件

- `src/device/hip/device_impl.h` / `src/device/hip/device_impl.cpp` — 父类 HIP 设备实现
- `src/device/hiprt/queue.h` / `src/device/hiprt/queue.cpp` — HIPRT 设备队列，负责内核排队与执行
- `src/bvh/hiprt.h` / `src/bvh/hiprt.cpp` — `BVHHIPRT` 数据结构，存储 HIPRT 专用的 BVH 数据
- `src/kernel/device/hiprt/globals.h` — `KernelParamsHIPRT` 结构体定义，GPU 端内核参数
- `src/device/hip/device.cpp` — HIP 设备工厂，根据配置选择创建 `HIPDevice` 或 `HIPRTDevice`
- `src/device/CMakeLists.txt` — 构建系统配置，条件编译 HIPRT 相关源文件
