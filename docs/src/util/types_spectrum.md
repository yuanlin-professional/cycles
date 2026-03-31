# types_spectrum.h - 光谱类型抽象层

## 概述

`types_spectrum.h` 为 Cycles 的光谱（颜色）计算提供类型抽象层。当前实现将光谱定义为 3 通道 RGB（`float3`），通过类型别名和宏定义将光谱操作映射到对应的 `float3` 操作。这种抽象设计使得未来扩展到光谱渲染（如更多波长通道）时只需修改此文件，而无需改动内核其他部分。

## 类与结构体

本文件不定义新的结构体，仅提供类型别名：

| 类型别名 | 映射目标 | 说明 |
|----------|----------|------|
| `Spectrum` | `float3` | 光谱值类型（3 通道 RGB） |
| `PackedSpectrum` | `packed_float3` | 紧凑存储的光谱值（12 字节） |

**常量宏**：
- `SPECTRUM_CHANNELS = 3` — 光谱通道数

## 核心函数/运算符重载

所有函数均通过宏映射到 `float3` 的对应函数：

| 宏定义 | 映射目标 | 功能说明 |
|--------|----------|----------|
| `make_spectrum(f)` | `make_float3(f)` | 从标量构造光谱值 |
| `load_spectrum(f)` | `load_float3(f)` | 加载光谱值 |
| `store_spectrum(s, f)` | `store_float3(f)` | 存储光谱值 |
| `zero_spectrum` | `zero_float3` | 零光谱值 |
| `one_spectrum` | `one_float3` | 全 1 光谱值 |

**遍历宏**：
- `FOREACH_SPECTRUM_CHANNEL(counter)` — 遍历所有光谱通道的 for 循环宏
- `GET_SPECTRUM_CHANNEL(v, i)` — 按索引访问光谱通道值

## 依赖关系

- **内部头文件**:
  - `util/types_float3.h`
- **被引用**: `util/types.h`、`kernel/util/colorspace.h`、`kernel/closure/bsdf_util.h`

## 关联文件

- `src/util/types_float3.h` — 底层 `float3` 类型定义
- `src/kernel/util/colorspace.h` — 色彩空间转换
- `src/kernel/closure/bsdf_util.h` — BSDF 闭包中使用光谱类型
