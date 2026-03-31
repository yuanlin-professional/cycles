# types.h - Cycles 基础数据类型聚合头文件

## 概述

`types.h` 是 Cycles 渲染器类型系统的顶层聚合头文件，通过统一的 `#include` 将所有基础数值类型（整数、浮点、无符号整数、颜色、光谱等）汇集到一个入口点。其他模块只需包含此单一头文件即可获得完整的类型定义。该文件本身不包含任何类型实现，仅作为类型子模块的索引和分发枢纽。

## 核心函数/类

`types.h` 本身不定义函数或类，但通过包含的子头文件提供以下类型体系：

### 基础类型（`types_base.h`）
- `uchar`, `uint`, `ushort` — 无符号整数类型别名
- `device_ptr` — 设备内存指针类型（`uint64_t`）
- `align_up()` — 内存对齐辅助函数

### 无符号字符向量
- `uchar2` (`types_uchar2.h`) — 2 分量无符号字节向量
- `uchar3` (`types_uchar3.h`) — 3 分量无符号字节向量
- `uchar4` (`types_uchar4.h`) — 4 分量无符号字节向量

### 有符号整数向量
- `int2` (`types_int2.h`) — 2 分量整数向量
- `int3` (`types_int3.h`) — 3 分量整数向量
- `int4` (`types_int4.h`) — 4 分量整数向量
- `int8` (`types_int8.h`) — 8 分量整数向量（用于 AVX 等宽 SIMD）

### 无符号整数向量
- `uint2` (`types_uint2.h`) — 2 分量无符号整数向量
- `uint3` (`types_uint3.h`) — 3 分量无符号整数向量
- `uint4` (`types_uint4.h`) — 4 分量无符号整数向量

### 无符号短整数向量
- `ushort4` (`types_ushort4.h`) — 4 分量无符号短整数向量

### 浮点向量
- `float2` (`types_float2.h`) — 2 分量浮点向量
- `float3` (`types_float3.h`) — 3 分量浮点向量（核心 3D 向量类型）
- `float4` (`types_float4.h`) — 4 分量浮点向量（含 SSE `__m128` 支持）
- `float8` (`types_float8.h`) — 8 分量浮点向量（AVX `__m256` 支持）

### 特殊类型
- `rgbe` (`types_rgbe.h`) — RGBE HDR 编码类型
- `Spectrum` (`types_spectrum.h`) — 光谱颜色表示类型
- `dual`, `dual3`, `dual4` (`types_dual.h`) — 自动微分的对偶数类型，用于微分光线追踪

## 依赖关系

- **内部头文件**: 包含以下 17 个类型子头文件：
  - `util/types_base.h` — 基础类型和宏定义
  - `util/types_uchar2.h` ~ `util/types_uchar4.h` — 无符号字节向量
  - `util/types_int2.h` ~ `util/types_int8.h` — 有符号整数向量
  - `util/types_uint2.h` ~ `util/types_uint4.h` — 无符号整数向量
  - `util/types_ushort4.h` — 无符号短整数向量
  - `util/types_float2.h` ~ `util/types_float8.h` — 浮点向量
  - `util/types_rgbe.h` — RGBE 编码
  - `util/types_spectrum.h` — 光谱类型
  - `util/types_dual.h` — 对偶数类型
- **被引用**: 作为 Cycles 最基础的类型头文件之一，被 56 个以上的文件引用，覆盖几乎所有子系统：
  - 内核层: 所有设备兼容层（`cuda/compat.h`, `hip/compat.h`, `metal/compat.h`, `optix/compat.h`, `oneapi/compat.h`）
  - 工具层: `util/math.h`, `util/transform.h`, `util/hash.h`, `util/color.h`, `util/boundbox.h` 等
  - 场景层: `scene/camera.h`, `scene/object.h`, `scene/geometry.h`, `scene/mesh.h` 等
  - BVH 层: `bvh/node.h`, `bvh/bvh.h`, `bvh/binning.h` 等
  - 设备层: `device/memory.h`, `device/device.h` 等

## 实现细节

1. **IWYU pragma**: 每个 `#include` 都标注了 `// IWYU pragma: export`，指示 Include-What-You-Use 工具将这些间接包含视为 `types.h` 的公开接口的一部分。包含 `types.h` 的文件不需要再单独包含子头文件
2. **跨平台兼容**: 子头文件中的向量类型在不同平台上有不同的底层实现：
   - CPU 端: 可使用 SSE/AVX 内部函数（如 `float4` 可包含 `__m128`）
   - GPU 端（CUDA/HIP/Metal/OneAPI）: 使用各平台原生向量类型或结构体
3. **类型命名约定**: 采用 OpenCL 风格的类型命名（`float2`, `int3`, `uint4` 等），在 CUDA 和 C++ 中通过 typedef/struct 兼容
4. **无循环依赖**: `types.h` 位于依赖图的底层，不依赖任何高级别模块（仅依赖 `defines.h`、`optimization.h`、`simd.h` 等更底层的配置头文件）

## 关联文件

- `src/util/types_base.h` — 基础标量类型和宏定义
- `src/util/types_float3.h` — 核心 3D 向量类型，广泛用于位置、方向、颜色
- `src/util/types_float4.h` — 4D 向量类型，包含 SSE 优化支持
- `src/util/types_dual.h` — 对偶数类型，用于微分渲染
- `src/util/types_spectrum.h` — 光谱类型定义
- `src/util/math.h` — 基于这些类型的数学运算函数
- `src/util/defines.h` — 编译器/平台宏定义（`CCL_NAMESPACE_BEGIN` 等）
