# device_impl.h / device_impl.cpp - CUDA 设备核心实现

## 概述

本文件定义并实现了 `CUDADevice` 类，是 Cycles 渲染器 CUDA 后端的核心设备类。它继承自 `GPUDevice`，提供了完整的 CUDA 设备生命周期管理，包括上下文创建与销毁、内核编译与加载、各类内存管理（设备内存、全局内存、纹理内存、共享内存）以及图形互操作支持。该类也是 OptiX 后端的基类，被 `OptiXDevice` 进一步继承扩展。

## 类与结构体

### `CUDADevice`
- **继承**: `GPUDevice`（定义于 `device/device.h`）
- **功能**: 封装单个 CUDA 物理设备的所有操作，管理 CUDA 上下文、模块和内存
- **友元类**: `CUDAContextScope` — 允许上下文作用域管理器访问私有成员
- **关键成员**:
  - `cuDevice` (`CUdevice`) — CUDA 设备句柄
  - `cuContext` (`CUcontext`) — CUDA 主上下文
  - `cuModule` (`CUmodule`) — 已加载的 CUDA 内核模块
  - `pitch_alignment` (`int`) — 纹理内存的 pitch 对齐值
  - `cuDevId` (`int`) — CUDA 设备序号
  - `cuDevArchitecture` (`int`) — GPU 架构版本（如 `major*100 + minor*10`）
  - `first_error` (`bool`) — 标记是否为首次错误（用于输出帮助信息）
  - `kernels` (`CUDADeviceKernels`) — 内核函数缓存集合
- **关键方法**:
  - `have_precompiled_kernels()` — 静态方法，检查 `lib/` 路径下是否存在预编译内核
  - `get_bvh_layout_mask()` — 返回 BVH 布局掩码，CUDA 后端固定使用 `BVH_LAYOUT_BVH2`
  - `compile_kernel_get_common_cflags()` — 生成 NVCC 通用编译参数
  - `compile_kernel()` — 编译或定位 CUDA 内核二进制文件（cubin/ptx）
  - `load_kernels()` — 加载内核模块到 GPU，调用 `kernels.load()` 初始化所有内核函数
  - `reserve_local_memory()` — 预分配局部内存以预测显存使用
  - `mem_alloc()` / `mem_copy_to()` / `mem_copy_from()` / `mem_zero()` / `mem_free()` — 通用设备内存管理接口
  - `global_alloc()` / `global_copy_to()` / `global_free()` — 全局内存管理（带常量指针更新）
  - `tex_alloc()` / `tex_copy_to()` / `tex_free()` — CUDA 纹理对象管理
  - `shared_alloc()` / `shared_free()` / `shared_to_device_pointer()` — 主机映射共享内存管理
  - `should_use_graphics_interop()` — 判断是否使用 OpenGL/Vulkan 图形互操作
  - `gpu_queue_create()` — 创建 `CUDADeviceQueue` 实例
  - `get_num_multiprocessors()` / `get_max_num_threads_per_multiprocessor()` — 查询 GPU 硬件参数

## 核心函数

### 构造函数 `CUDADevice::CUDADevice()`
初始化流程：
1. 验证 `texMemObject` 与 `CUtexObject` 的类型大小一致性
2. 调用 `cuInit(0)` 初始化 CUDA
3. 通过 `cuDeviceGet()` 获取设备句柄
4. 检测主机内存映射能力（`CU_DEVICE_ATTRIBUTE_CAN_MAP_HOST_MEMORY`）
5. 配置并保留 CUDA 主上下文（使用 `CU_CTX_LMEM_RESIZE_TO_MAX` 标志）
6. 计算 GPU 架构版本号

### `compile_kernel()`
内核编译查找策略（按优先级）：
1. 非自适应编译模式下，查找预编译 cubin（`lib/{name}_sm_{major}{minor}.cubin.zst`）
2. 查找预编译 PTX（`lib/{name}_compute_{major}{minor}.ptx.zst`），向下兼容到 SM 5.0
3. 查找本地缓存的已编译内核（路径含源码 MD5 + 编译标志 MD5）
4. 调用 `nvcc` 实时编译

### `tex_alloc()`
纹理分配流程：
1. 根据纹理扩展模式设置 CUDA 地址模式（Wrap/Clamp/Border/Mirror）
2. 根据插值模式设置过滤模式（Point/Linear）
3. 根据数据类型选择 CUDA 数组格式（UINT8/UINT16/FLOAT/HALF）
4. 2D 纹理使用 pitch 对齐的线性内存，1D 纹理使用普通线性内存
5. 非 NanoVDB 纹理创建 `CUtexObject` 绑定无指针纹理对象
6. 更新全局纹理信息表

### `should_use_graphics_interop()`
OpenGL 互操作：通过 `cuGLGetDevices()` 检查当前 CUDA 设备是否在 OpenGL 上下文中。
Vulkan 互操作：通过 `cuDeviceGetUuid()` 比较设备 UUID 进行匹配。

## 依赖关系

- **内部头文件**:
  - `device/cuda/kernel.h` — `CUDADeviceKernels` 类
  - `device/cuda/queue.h` — `CUDADeviceQueue` 类
  - `device/cuda/util.h` — `CUDAContextScope`、`cuda_assert` 宏
  - `device/device.h` — `GPUDevice` 基类
  - `kernel/device/cuda/globals.h` — `KernelParamsCUDA` 结构体
  - `session/display_driver.h` — 显示驱动接口
  - `util/debug.h`、`util/log.h`、`util/md5.h`、`util/path.h`、`util/string.h`、`util/system.h`、`util/texture.h`、`util/time.h`、`util/types.h`
- **被引用**:
  - `src/device/cuda/device.cpp` — 设备工厂函数创建 `CUDADevice` 实例
  - `src/device/cuda/kernel.cpp` — 内核加载使用 `CUDADevice::cuModule`
  - `src/device/cuda/queue.cpp` — 命令队列引用 `CUDADevice`
  - `src/device/cuda/graphics_interop.cpp` — 图形互操作引用 `CUDADevice`
  - `src/device/cuda/util.cpp` — 上下文作用域使用 `CUDADevice::cuContext`
  - `src/device/optix/device_impl.h` — `OptiXDevice` 继承自 `CUDADevice`

## 实现细节 / 关键算法

1. **主上下文管理**: 使用 `cuDevicePrimaryCtxRetain/Release` 而非 `cuCtxCreate/Destroy`，确保在同一设备上多次创建 `CUDADevice` 时共享上下文，避免与 OptiX 等库冲突。
2. **局部内存预分配**: `reserve_local_memory()` 通过启动一次最大内核（根据特性选择 `SHADE_SURFACE_RAYTRACE` / `SHADE_SURFACE_MNEE` / `SHADE_SURFACE`）来触发 CUDA 驱动预分配局部内存，配合 `CU_CTX_LMEM_RESIZE_TO_MAX` 标志确保后续内核启动不会意外分配额外显存。
3. **内核编译缓存**: 使用源文件 MD5 和编译标志的组合哈希作为缓存键，确保源码或编译选项变化时重新编译。
4. **P2P 对等访问**: `check_peer_access()` 不仅检查基本的对等访问能力，还验证 CUDA 数组跨设备访问（用于 3D 纹理），并双向启用对等访问。
5. **常量内存更新**: `const_copy_to()` 通过 `cuModuleGetGlobal()` 获取 `kernel_params` 的设备端地址，然后使用 `cuMemcpyHtoD()` 更新对应字段偏移处的数据。使用 `kernel/data_arrays.h` 的宏展开机制自动匹配字段名称。

## 关联文件

- `src/device/cuda/device.h` / `device.cpp` — CUDA 设备注册入口
- `src/device/cuda/kernel.h` / `kernel.cpp` — 内核函数管理
- `src/device/cuda/queue.h` / `queue.cpp` — 命令队列
- `src/device/cuda/graphics_interop.h` / `graphics_interop.cpp` — 图形互操作
- `src/device/cuda/util.h` / `util.cpp` — CUDA 工具类
- `src/device/optix/device_impl.h` — OptiX 后端继承本类
- `src/kernel/device/cuda/globals.h` — CUDA 内核全局参数定义
