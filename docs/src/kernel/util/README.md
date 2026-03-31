# kernel/util - 内核工具函数

## 概述

`kernel/util/` 提供在 GPU 和 CPU 内核中共用的工具函数和辅助模块。这些工具涵盖色彩空间转换、光线微分计算、查找表插值、IES 灯光配置文件解析、NanoVDB 体积数据访问、性能分析宏以及 3D 纹理插值等功能。所有代码以 GPU 兼容的内联头文件形式编写，通过 `ccl_device` / `ccl_device_inline` 修饰符保证跨平台编译。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `colorspace.h` | 头文件 | 色彩空间转换工具：XYZ 到 RGB、Rec709 到渲染色彩空间、线性 RGB 到灰度、Spectrum 与 RGB 互转 |
| `differential.h` | 头文件 | 光线微分（Ray Differential）传递与计算：`differential_transfer`（通过均匀介质传递 dP）、`differential_incoming`（入射方向微分）、`differential_dudv`（UV 微分） |
| `ies.h` | 头文件 | IES 灯光配置文件采样：球面坐标插值查找，支持三次样条插值和边界环绕处理 |
| `lookup_table.h` | 头文件 | 通用查找表插值：1D/2D/3D 线性插值读取函数，用于从内核数据数组中采样预计算的表格数据 |
| `nanovdb.h` | 头文件 | NanoVDB 体积数据结构的内核端最小化实现：Grid、Tree、Node 层级结构和体素读取。从 OpenVDB NanoVDB 头文件精简而来，添加了 Metal 地址空间限定符兼容 |
| `profiler.h` | 头文件 | 内核性能分析宏：`PROFILING_INIT`、`PROFILING_EVENT`、`PROFILING_SHADER`。仅在 CPU 端有效，GPU 端为空操作 |
| `texture_3d.h` | 头文件 | 3D 纹理/体积插值工具：基于 NanoVDB 网格的三次和随机插值（`interp_tricubic_stochastic`），支持 GPU 与 CPU 的模板特化 |

## 核心类与数据结构

### differential3 / differential
光线微分结构体，存储 `dx` 和 `dy` 偏导数。`differential3` 用于三维空间微分（位置 dP、方向 dD），`differential` 用于标量微分（UV 坐标 du、dv）。这些微分用于纹理过滤（MIP 贴图级别选择）和着色中的精确导数计算。

### NanoVDB 数据结构（nanovdb.h）
OpenVDB NanoVDB 库的内核端精简版本：
- **`Grid<TreeT>`** - 体积网格顶层容器，包含元数据和根指针
- **`Tree<RootT>`** - 树结构，管理节点层级
- **`InternalNode<ChildT, LOG2DIM>`** - 内部节点（稀疏体素八叉树）
- **`LeafNode<ValueT>`** - 叶节点，存储实际体素值
- **`ReadAccessor`** - 缓存优化的体素读取器

### FresnelThinFilm（在 closure/bsdf_util.h 中定义，此处相关）
薄膜干涉参数结构体，被色彩空间工具间接支持。

## 内核函数入口

本目录不包含独立的内核入口点。所有函数均为内联辅助函数，被其他内核模块调用：

| 函数 | 文件 | 说明 |
|------|------|------|
| `xyz_to_rgb()` | `colorspace.h` | XYZ 色彩空间到渲染色彩空间转换 |
| `rec709_to_rgb()` | `colorspace.h` | Rec709 到渲染色彩空间转换 |
| `linear_rgb_to_gray()` | `colorspace.h` | 线性 RGB 到亮度值 |
| `rgb_to_spectrum()` / `spectrum_to_rgb()` | `colorspace.h` | RGB 与光谱表示互转 |
| `differential_transfer()` | `differential.h` | 光线微分通过均匀介质传递到着色点 |
| `differential_incoming()` | `differential.h` | 计算入射光线的方向微分 |
| `differential_dudv()` | `differential.h` | 从光线微分计算 UV 参数化微分 |
| `kernel_ies_interp()` | `ies.h` | IES 灯光配置文件球面坐标插值 |
| `lookup_table_read()` | `lookup_table.h` | 1D 查找表线性插值 |
| `lookup_table_read_2D()` | `lookup_table.h` | 2D 查找表双线性插值 |
| `lookup_table_read_3D()` | `lookup_table.h` | 3D 查找表三线性插值 |
| `interp_tricubic_stochastic()` | `texture_3d.h` | 随机三次体积插值（单次采样） |

## GPU 兼容性

- **NanoVDB (`nanovdb.h`)**: 从原始 NanoVDB 头文件裁剪，添加了 Metal 地址空间限定符（`ccl_global`），解决了原始头文件与 Metal 不兼容的问题。在 Metal 和 oneAPI 后端上通过 `WITH_NANOVDB` 条件编译控制。
- **性能分析 (`profiler.h`)**: 仅在 CPU 端有效（`#ifndef __KERNEL_GPU__`），GPU 端所有 `PROFILING_*` 宏展开为空。
- **3D 纹理 (`texture_3d.h`)**: 使用模板匿名命名空间避免不同指令集内核之间的符号冲突（CPU 端 SSE/AVX 特化）。

## 依赖关系

### 上游依赖（本模块依赖）
- `util/` (顶层) - 基础数学函数（`math.h`、`math_fast.h`）、类型定义（`types.h`、`types_spectrum.h`）、投影工具（`projection.h`）、哈希函数（`hash.h`）、颜色工具（`color.h`）、纹理工具（`texture.h`）
- `kernel/globals.h` - 内核全局数据访问（`KernelGlobals`、`kernel_data`）
- `kernel/types.h` - 内核类型定义
- `kernel/tables.h` - 常量查找表数据
- `kernel/sample/lcg.h` - 线性同余随机数生成器（用于 3D 纹理随机采样）

### 下游依赖（依赖本模块）
- `kernel/closure/` - BSDF 闭包使用查找表和色彩空间转换
- `kernel/camera/` - 相机模块使用微分计算和查找表
- `kernel/integrator/` - 积分器使用性能分析宏和色彩转换
- `kernel/bake/` - 烘焙模块使用色彩空间转换
- `kernel/light/` - 灯光模块使用 IES 采样
- `kernel/svm/` - SVM 节点使用查找表和色彩转换
- `kernel/geom/` - 几何体模块使用微分计算

## 关键算法与实现细节

### 光线微分传递（differential.h）
基于 Homan Igehy 1999 年论文 "Tracing Ray Differentials" 的实现。光线微分 `dP` 和 `dD` 在光线沿着路径传播时被传递到着色点。`differential_transfer` 函数将光线微分通过均匀介质传递，`differential_dudv` 将空间微分投影到 UV 参数空间（选择最稳定的投影轴以避免数值退化）。

### NanoVDB 内核端访问（nanovdb.h）
NanoVDB 使用分层稀疏体素结构（类似八叉树）存储体积数据。内核端实现了只读访问路径：`ReadAccessor` 通过缓存最近访问的节点来加速体素查找，避免每次从根节点遍历。支持 `float`、`float3`（`nanovdb::Vec3f`）等数据类型。

### 随机三次体积插值（texture_3d.h）
基于 "Stochastic Texture Filtering"（2023）论文的单次采样三次插值。使用蓄水池采样从 4x4x4 三次权重中随机选择一个采样点，将三次插值的开销降低为一次查找，适用于 GPU 上的高效体积渲染。

## 参见

- `src/util/` - 基础工具库（数学、类型、哈希、颜色）
- `src/kernel/closure/bsdf_util.h` - 菲涅尔和薄膜干涉工具
- `src/kernel/sample/` - 采样模式和随机数生成
- `src/kernel/device/cpu/image.h` - CPU 端图像纹理实现
