# util - 基础工具库

## 概述

util 是 Cycles 渲染器的基础工具库，提供数学运算、向量类型、数据结构、线程管理、内存分配、字符串处理、图像操作等底层功能。作为 Cycles 架构中最底层的模块，它不依赖任何 Cycles 上层模块，而被所有其他模块广泛使用。

该模块的设计兼顾 CPU 和 GPU 运行环境——许多头文件通过 `__KERNEL_GPU__` 宏提供在 CUDA、HIP、Metal 等 GPU 编程语言中可编译的实现，使得同一套数学和类型定义可以在主机端和设备端共享。

## 目录结构

### 数学与类型

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `types.h` | 头文件 | 聚合头文件，统一包含所有基础类型定义 |
| `types_base.h` | 头文件 | 基础标量类型定义（`ccl_device_inline` 等宏） |
| `types_float2.h` | 头文件 | `float2` 二维浮点向量类型 |
| `types_float3.h` | 头文件 | `float3` 三维浮点向量类型（核心渲染数据类型） |
| `types_float4.h` | 头文件 | `float4` 四维浮点向量类型（含 SSE 优化） |
| `types_float8.h` | 头文件 | `float8` 八维浮点类型（AVX 支持） |
| `types_int2.h` | 头文件 | `int2` 二维整数类型 |
| `types_int3.h` | 头文件 | `int3` 三维整数类型 |
| `types_int4.h` | 头文件 | `int4` 四维整数类型 |
| `types_int8.h` | 头文件 | `int8` 八维整数类型 |
| `types_uint2.h` | 头文件 | `uint2` 二维无符号整数类型 |
| `types_uint3.h` | 头文件 | `uint3` 三维无符号整数类型 |
| `types_uint4.h` | 头文件 | `uint4` 四维无符号整数类型 |
| `types_uchar2.h` | 头文件 | `uchar2` 二维无符号字节类型 |
| `types_uchar3.h` | 头文件 | `uchar3` 三维无符号字节类型 |
| `types_uchar4.h` | 头文件 | `uchar4` 四维无符号字节类型 |
| `types_ushort4.h` | 头文件 | `ushort4` 四维无符号短整数类型 |
| `types_spectrum.h` | 头文件 | 光谱颜色类型定义 |
| `types_rgbe.h` | 头文件 | RGBE（HDR）编码类型 |
| `types_dual.h` | 头文件 | 对偶数类型（用于自动微分） |
| `math.h` | 头文件 | 聚合数学头文件 |
| `math_base.h` | 头文件 | 基础数学函数（min/max、clamp、lerp 等） |
| `math_float2.h` | 头文件 | `float2` 数学运算 |
| `math_float3.h` | 头文件 | `float3` 数学运算（点积、叉积、归一化等） |
| `math_float4.h` | 头文件 | `float4` 数学运算 |
| `math_float8.h` | 头文件 | `float8` 数学运算 |
| `math_int2.h` | 头文件 | `int2` 数学运算 |
| `math_int3.h` | 头文件 | `int3` 数学运算 |
| `math_int4.h` | 头文件 | `int4` 数学运算 |
| `math_int8.h` | 头文件 | `int8` 数学运算 |
| `math_dual.h` | 头文件 | 对偶数运算（微分几何） |
| `math_fast.h` | 头文件 | 快速近似数学函数（fast_sinf、fast_cosf 等） |
| `math_intersect.h` | 头文件 | 几何求交数学（射线-三角形、射线-包围盒等） |
| `math_cdf.h` / `.cpp` | 头文件/源文件 | 累积分布函数 (CDF) 工具 |
| `transform.h` / `.cpp` | 头文件/源文件 | 仿射变换矩阵（`Transform` 4×3 矩阵） |
| `transform_avx2.cpp` | 源文件 | AVX2 优化的变换运算 |
| `projection.h` | 头文件 | 投影变换（4×4 矩阵） |
| `projection_inverse.h` | 头文件 | 投影矩阵求逆 |
| `boundbox.h` | 头文件 | 轴对齐包围盒 (`BoundBox`) |
| `rect.h` | 头文件 | 二维矩形区域 |

### 容器与数据结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `array.h` | 头文件 | 动态数组（GPU 可用） |
| `vector.h` | 头文件 | `std::vector` 封装 |
| `map.h` | 头文件 | `std::map` / `std::unordered_map` 封装 |
| `set.h` | 头文件 | `std::set` / `std::unordered_set` 封装 |
| `list.h` | 头文件 | `std::list` 封装 |
| `deque.h` | 头文件 | `std::deque` 封装 |
| `queue.h` | 头文件 | 队列容器 |
| `stack_allocator.h` | 头文件 | 栈上分配器（避免小对象堆分配） |
| `hash.h` | 头文件 | 哈希函数工具集 |
| `murmurhash.h` / `.cpp` | 头文件/源文件 | MurmurHash 哈希算法实现 |
| `md5.h` / `.cpp` | 头文件/源文件 | MD5 摘要算法实现 |
| `disjoint_set.h` | 头文件 | 并查集数据结构 |
| `concurrent_set.h` | 头文件 | 线程安全集合（TBB） |
| `concurrent_vector.h` | 头文件 | 线程安全向量（TBB） |
| `algorithm.h` | 头文件 | 算法工具（排序等） |
| `unique_ptr.h` | 头文件 | `std::unique_ptr` 封装 |
| `unique_ptr_vector.h` | 头文件 | 唯一指针向量容器 |

### 线程与并发

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `thread.h` / `.cpp` | 头文件/源文件 | 线程类（自定义栈大小支持） |
| `task.h` / `.cpp` | 头文件/源文件 | TBB 任务并行封装 |
| `tbb.h` | 头文件 | TBB (Threading Building Blocks) 头文件 |
| `atomic.h` | 头文件 | 原子操作 |
| `semaphore.h` | 头文件 | 信号量 |

### 字符串与 I/O

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `string.h` / `.cpp` | 头文件/源文件 | 字符串工具（格式化、转换等） |
| `path.h` / `.cpp` | 头文件/源文件 | 文件路径操作 |
| `log.h` / `.cpp` | 头文件/源文件 | 日志系统（基于 Google glog） |
| `args.h` | 头文件 | 命令行参数解析 |
| `param.h` | 头文件 | 参数字符串处理 |

### 图像与颜色

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `color.h` | 头文件 | 颜色空间转换（sRGB、线性等） |
| `half.h` | 头文件 | 半精度浮点数 (FP16) 支持 |
| `image.h` | 头文件 | 图像数据结构与操作 |
| `image_impl.h` | 头文件 | 图像操作的模板实现 |
| `texture.h` | 头文件 | 纹理数据类型与插值 |
| `ies.h` / `.cpp` | 头文件/源文件 | IES 光度学配光曲线解析 |
| `nanovdb.h` / `.cpp` | 头文件/源文件 | NanoVDB 体积数据支持 |
| `openvdb.h` / `.cpp` | 头文件/源文件 | OpenVDB 体积数据支持 |
| `openimagedenoise.h` | 头文件 | Intel Open Image Denoise 集成接口 |

### 内存管理

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `aligned_malloc.h` / `.cpp` | 头文件/源文件 | 对齐内存分配（SSE/AVX 要求） |
| `guarded_allocator.h` / `.cpp` | 头文件/源文件 | 带追踪的内存分配器（调试用） |

### 系统与平台

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `system.h` / `.cpp` | 头文件/源文件 | 系统信息（CPU 特性检测、内存查询） |
| `windows.h` / `.cpp` | 头文件/源文件 | Windows 平台适配（UTF-8 路径等） |
| `version.h` | 头文件 | Cycles 版本号定义 |
| `defines.h` | 头文件 | 全局宏定义（`CCL_NAMESPACE_BEGIN` 等） |
| `optimization.h` | 头文件 | 编译器优化提示（SSE/AVX 指令集选择） |
| `simd.h` | 头文件 | SIMD 指令集抽象层 |
| `static_assert.h` | 头文件 | 静态断言工具 |

### 其他工具

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `debug.h` / `.cpp` | 头文件/源文件 | 调试工具与标志 |
| `profiling.h` / `.cpp` | 头文件/源文件 | 性能剖析工具 |
| `progress.h` | 头文件 | 渲染进度报告 |
| `stats.h` | 头文件 | 统计信息收集 |
| `guiding.h` | 头文件 | Intel Open Path Guiding Library 集成 |
| `CMakeLists.txt` | 构建文件 | CMake 构建配置 |

## 核心类与数据结构

### float2 / float3 / float4 / float8
- 定义位置: `types_float2.h`, `types_float3.h`, `types_float4.h`, `types_float8.h`
- 功能: Cycles 核心向量类型，在渲染管线中无处不在
- 关键特性: 支持 SSE/AVX SIMD 加速，GPU 兼容（通过 `__KERNEL_GPU__` 宏切换实现）

### Transform
- 定义位置: `transform.h`
- 功能: 4×3 仿射变换矩阵，存储为三个 `float4` 行
- 关键方法: `transform_point()`, `transform_direction()`, `transform_inverse()`
- 设计: 紧凑存储，比 4×4 矩阵节省内存；包含 `DecomposedTransform` 用于运动模糊插值

### BoundBox
- 定义位置: `boundbox.h`
- 功能: 轴对齐包围盒 (AABB)，由 `min` 和 `max` 两个 `float3` 定义
- 关键方法: `grow()`, `intersect()`, `center()`, `area()`
- 用途: 层次包围体 (BVH) 构建和光线求交加速

### thread / thread_mutex / thread_scoped_lock
- 定义位置: `thread.h`
- 功能: 线程原语封装，支持自定义栈大小（macOS 需要）
- 关键类型: `thread_mutex` (= `std::mutex`), `thread_scoped_lock` (= `std::unique_lock`), `thread_spin_lock` (= `tbb::spin_mutex`)

### TaskPool / TaskScheduler
- 定义位置: `task.h`
- 功能: 基于 TBB 的任务并行框架
- 用途: 并行 BVH 构建、并行图像处理、并行场景更新

### array\<T\>
- 定义位置: `array.h`
- 功能: 动态数组，支持设备内存同步（GPU 数据传输标记）
- 特性: 比 `std::vector` 更适合渲染器场景，支持对齐分配

## 模块架构

util 模块采用扁平的头文件组织结构，不存在复杂的类继承层次。核心设计原则：

1. **GPU/CPU 双重兼容**: 大量头文件通过 `__KERNEL_GPU__` 条件编译，同时支持 CPU 和 GPU 编译路径
2. **SIMD 透明优化**: 向量类型在支持 SSE/AVX 的平台上自动使用 SIMD 指令
3. **零开销抽象**: 使用 `ccl_device_inline` 等宏确保关键路径函数内联
4. **标准库封装**: 容器类型通常是标准库容器的薄封装，主要添加 Cycles 命名空间和分配器支持

## 依赖关系

### 上游依赖（本模块依赖）
- **C++ 标准库**: `<mutex>`, `<thread>`, `<vector>`, `<string>` 等
- **TBB (Threading Building Blocks)**: 任务并行和并发容器
- **OpenImageIO**: 图像 I/O（间接）
- **NanoVDB / OpenVDB**: 体积数据（可选）
- **Intel OIDN**: 降噪（可选，`openimagedenoise.h`）
- **Intel OPG**: 路径引导（可选，`guiding.h`）

### 下游依赖（依赖本模块）
- **Cycles 所有其他模块**: util 是整个 Cycles 的底层基础，被 `graph/`, `subd/`, `bvh/`, `device/`, `kernel/`, `scene/`, `session/`, `integrator/`, `hydra/`, `app/` 等所有模块依赖

## 关键算法与实现细节

### SIMD 向量运算
`float3`/`float4` 在 x86 平台上使用 `__m128` SSE 内建类型，所有基本运算（加减乘除、点积、叉积）编译为单条 SIMD 指令。`float8` 使用 `__m256` AVX 类型。

### 快速数学近似
`math_fast.h` 提供一系列快速近似函数（`fast_sinf`, `fast_cosf`, `fast_expf`, `fast_logf` 等），用于着色器计算中精度要求不高但性能敏感的场景。

### 对偶数自动微分
`types_dual.h` 和 `math_dual.h` 实现了对偶数（dual number），用于着色器中纹理坐标微分的自动计算，支持 Mipmap 级别选择和各向异性过滤。

### MurmurHash 哈希
`murmurhash.h` 实现了 MurmurHash3 算法，用于着色器中的确定性随机数生成和纹理哈希。

## 参见

- [graph - 节点图系统](../graph/README.md)
- [kernel - 渲染内核](../kernel/README.md)（共享数学类型和宏定义）
- [device - 设备抽象层](../device/README.md)（使用线程和内存管理）
