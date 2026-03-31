# image.h - 内核图像纹理访问入口

## 概述

`image.h` 是一个轻量级桥接头文件，用于在内核代码中统一引入图像纹理采样功能。与 `globals.h` 类似，它根据编译目标平台选择性地包含对应的设备端图像实现。对于 GPU 设备，图像纹理函数在各自的设备头文件中预先定义；对于 CPU 设备，本文件引入 `kernel/device/cpu/image.h`，提供基于软件的纹理插值和采样实现。

## 类与结构体

本文件自身不定义任何类或结构体。通过包含的设备端头文件，引入了图像纹理采样相关的函数接口，通常包括：
- 2D/3D 纹理查找函数
- 支持各种插值模式（最近邻、线性、三次）
- 支持各种边界处理模式（重复、裁剪、延伸）

## 枚举与常量

无。

## 核心函数

无直接函数定义。具体的纹理采样函数由各设备端头文件提供：
- CPU 端: `kernel/device/cpu/image.h` — 提供基于模板的软件纹理采样
- GPU 端: 由各 GPU 平台的原生纹理采样硬件指令支持

## 依赖关系

- **内部头文件**:
  - `kernel/device/cpu/image.h` — CPU 端图像纹理采样实现（仅非 GPU 编译时）
- **被引用**:
  - `kernel/svm/image.h` — SVM 图像纹理节点
  - `kernel/svm/sky.h` — SVM 天空纹理节点

## 实现细节 / 关键算法

本文件与 `globals.h` 采用相同的平台抽象策略：

```
#ifndef __KERNEL_GPU__
  -> 包含 CPU 端 image.h
#endif
// GPU 端：纹理采样函数已由设备编译环境预定义
```

注释明确指出此设计同时服务于 clangd 代码分析工具，确保在编辑器环境中（默认以 CPU 模式解析）能正确识别纹理相关的函数和类型定义。

图像纹理在 Cycles 中的完整数据流为：
1. 场景端通过 `TextureInfo` 结构体（定义在 `data_arrays.h` 中的 `texture_info` 数组）描述每个纹理的元信息
2. CPU 端通过 `kernel/device/cpu/image.h` 中的函数按需从内存中采样
3. GPU 端通过原生纹理硬件（如 CUDA 纹理对象、Metal 纹理）进行硬件加速采样

## 关联文件

- `kernel/device/cpu/image.h` — CPU 端纹理采样的实际实现
- `kernel/device/gpu/image.h` — GPU 端纹理采样实现
- `kernel/globals.h` — 同类型的平台抽象桥接文件（用于 KernelGlobals）
- `kernel/svm/image.h` — 着色器虚拟机(SVM)中的图像纹理节点，是本文件的主要消费者
- `kernel/data_arrays.h` — 声明了 `texture_info` 数组，存储纹理元数据
