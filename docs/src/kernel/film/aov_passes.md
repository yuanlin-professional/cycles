# aov_passes.h - 自定义 AOV 渲染通道写入

## 概述

本文件实现了 Cycles 渲染器中任意输出变量（AOV, Arbitrary Output Variables）渲染通道的写入功能。AOV 允许用户在着色器中自定义输出数据，将标量值或颜色值写入独立的渲染通道，供后期合成使用。

文件提供两个函数，分别用于写入标量类型和颜色类型的 AOV 通道。

## 类与结构体

本文件未定义独立的类或结构体。

## 枚举与常量

本文件未定义枚举或常量。依赖 `kernel_data.film` 中的以下字段：

- `pass_aov_value` — AOV 标量通道在渲染缓冲区中的基础偏移量
- `pass_aov_color` — AOV 颜色通道在渲染缓冲区中的基础偏移量

## 核心函数

### film_write_aov_pass_value()
- **签名**: `ccl_device_inline void film_write_aov_pass_value(KernelGlobals kg, ConstIntegratorState state, ccl_global float *ccl_restrict render_buffer, const int aov_id, const float value)`
- **功能**: 将一个标量浮点值写入指定 AOV 通道。通过 `pass_aov_value + aov_id` 计算目标缓冲区偏移，使用 `film_write_pass_float()` 执行原子累加写入。

### film_write_aov_pass_color()
- **签名**: `ccl_device_inline void film_write_aov_pass_color(KernelGlobals kg, ConstIntegratorState state, ccl_global float *ccl_restrict render_buffer, const int aov_id, const float3 color)`
- **功能**: 将一个三通道颜色值写入指定 AOV 通道。颜色被转换为 `float4`（alpha 通道设为 1.0f），通过 `pass_aov_color + aov_id` 计算目标缓冲区偏移，使用 `film_write_pass_float4()` 执行原子累加写入。

## 依赖关系

- **内部头文件**:
  - `kernel/film/write.h` — 提供 `film_pass_pixel_render_buffer()`、`film_write_pass_float()`、`film_write_pass_float4()` 等底层渲染缓冲区写入函数

- **被引用**:
  - `src/kernel/svm/aov.h` — SVM（着色器虚拟机）中的 AOV 输出节点调用本文件的函数

## 实现细节 / 关键算法

AOV 通道的写入逻辑非常直接：

1. 通过 `film_pass_pixel_render_buffer()` 获取当前像素在渲染缓冲区中的基地址
2. 使用基地址加上 AOV 通道的偏移量定位目标位置
3. 调用底层写入函数（在 GPU 上使用原子操作，在 CPU 上使用普通累加）

AOV ID 由着色器编译阶段确定，每个 AOV 节点被分配唯一的 ID，用于在渲染缓冲区中定位各自的存储区域。

## 关联文件

- `src/kernel/film/write.h` — 渲染通道写入基础设施
- `src/kernel/svm/aov.h` — SVM 着色器中 AOV 节点的实现，是本文件函数的主要调用者
- `src/kernel/film/data_passes.h` — 其他数据通道的写入（深度、法线、UV 等）
