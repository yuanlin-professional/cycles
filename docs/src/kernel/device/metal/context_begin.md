# context_begin.h - MetalKernelContext 类定义开始

## 概述

本文件定义了 `MetalKernelContext` 类的开始部分，该类是 Metal 内核的上下文容器，封装了对 Metal 资源绑定（内核参数和辅助资源）的访问。它提供纹理采样适配器函数，使得 GPU 通用的图像采样代码可以在 Metal 后端正确运行。

本文件与 `context_end.h` 配对使用，中间插入 GPU 通用内核代码，形成完整的 `MetalKernelContext` 类定义。

## 核心函数/宏定义

### MetalKernelContext 类

```cpp
class MetalKernelContext {
  public:
    constant KernelParamsMetal &launch_params_metal;
    constant MetalAncillaries *metal_ancillaries;
};
```

**成员变量**:
- `launch_params_metal` — 内核启动参数的常量引用，包含所有全局数据数组和积分器状态
- `metal_ancillaries` — Metal 附加资源指针，包含纹理、加速结构和求交函数表

**构造函数**:
- `MetalKernelContext(KernelParamsMetal&, MetalAncillaries*)` — 完整构造（内核入口使用）
- `MetalKernelContext(KernelParamsMetal&)` — 仅参数构造（求交处理函数使用）

### 纹理采样适配器

| 函数 | 说明 |
|------|------|
| `ccl_gpu_tex_object_read_2D<T>()` (通用模板) | 默认实现，触发 `kernel_assert` 断言失败 |
| `ccl_gpu_tex_object_read_2D<float4>()` | 特化：采样 2D 纹理返回 `float4`，从纹理句柄解码纹理ID和采样器ID |
| `ccl_gpu_tex_object_read_2D<float>()` | 特化：采样 2D 纹理返回 `float`（取 `.x` 分量） |

### 纹理句柄编码

纹理对象句柄（`ccl_gpu_tex_object_2D`，即 `uint64_t`）编码方式：
- 低 32 位：纹理索引（`tid`）
- 高 32 位：采样器索引（`sid`）

通过 `metal_ancillaries->textures` 数组和 `metal_samplers` 预定义采样器进行实际采样。

## 依赖关系

- **内部头文件**:
  - `kernel/util/nanovdb.h` — NanoVDB 体积数据支持（条件编译 `WITH_NANOVDB`）
  - `kernel/device/gpu/image.h` — GPU 通用图像采样代码（在类内部包含）
- **被引用**:
  - `kernel/device/gpu/kernel.h` — 当 `__KERNEL_METAL__` 定义时包含本文件

## 实现细节 / 关键算法

### 上下文类包装模式

Metal 内核使用一种独特的"拆分类"模式：
1. `context_begin.h` 打开类定义，声明资源绑定和纹理适配器
2. `kernel/device/gpu/kernel.h` 中的 GPU 通用代码被包含在类内部，成为成员函数
3. `context_end.h` 关闭类定义

这使得 GPU 通用代码中通过 `this->` 隐式访问的 `launch_params_metal` 和 `metal_ancillaries` 在编译时可见。

### 纹理采样路径

```
ccl_gpu_tex_object_read_2D<float4>(tex, x, y)
  → 解码 tid = uint(tex), sid = uint(tex >> 32)
  → metal_ancillaries->textures[tid].tex.sample(metal_samplers[sid], float2(x,y))
```

`TextureParamsMetal` 与 `Texture2DParamsMetal` 通过内存重解释（reinterpret）关联。

## 关联文件

- `kernel/device/metal/context_end.h` — 配对文件，关闭 MetalKernelContext 类
- `kernel/device/metal/compat.h` — 定义 `MetalAncillaries` 结构体和采样器数组
- `kernel/device/metal/globals.h` — 定义 `KernelParamsMetal` 结构体
- `kernel/device/gpu/image.h` — GPU 通用图像处理代码
- `kernel/device/gpu/kernel.h` — GPU 通用内核框架
