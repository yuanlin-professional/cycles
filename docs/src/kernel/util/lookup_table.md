# lookup_table.h - 通用查找表插值读取

## 概述
本文件提供对预计算查找表（Lookup Table）进行线性插值读取的内核端工具函数，支持一维、二维和三维查找表。查找表数据存储在全局内存中，通过偏移量和尺寸参数定位。这些函数广泛用于 BSDF 闭包中的预计算积分表（如 GGX 能量补偿表）、相机光晕效果等需要快速查表的场景。

## 类与结构体
本文件未定义类或结构体。

## 枚举与常量
本文件未定义枚举或常量。

## 核心函数

### lookup_table_read()
- **签名**: `ccl_device float lookup_table_read(KernelGlobals kg, float x, const int offset, const int size)`
- **功能**: 一维查找表线性插值读取。将输入坐标 `x` 钳制到 [0, 1] 范围，映射到表索引后在相邻两个条目间进行线性插值。当插值因子恰好为 0 时跳过第二次读取以优化性能。

### lookup_table_read_2D()
- **签名**: `ccl_device float lookup_table_read_2D(KernelGlobals kg, const float x, float y, const int offset, const int xsize, const int ysize)`
- **功能**: 二维查找表双线性插值读取。在 y 维度上确定相邻两行，对每行分别调用 `lookup_table_read()` 完成 x 维度插值，最后在 y 维度上进行线性插值。

### lookup_table_read_3D()
- **签名**: `ccl_device float lookup_table_read_3D(KernelGlobals kg, const float x, float y, float z, const int offset, const int xsize, const int ysize, const int zsize)`
- **功能**: 三维查找表三线性插值读取。在 z 维度上确定相邻两个切片，对每个切片分别调用 `lookup_table_read_2D()` 完成 x-y 双线性插值，最后在 z 维度上进行线性插值。

## 依赖关系
- **内部头文件**:
  - `kernel/globals.h` — 提供 `KernelGlobals` 及 `kernel_data_fetch` 数据访问宏
  - `kernel/types.h` — 内核基础类型
- **被引用**:
  - `kernel/closure/bsdf_util.h` — BSDF 实用工具，用于能量补偿等预计算表的查询
  - `kernel/closure/bsdf_microfacet.h` — 微面元 BSDF（如 GGX）的预计算查找表
  - `kernel/closure/bsdf_sheen.h` — Sheen BSDF 的预计算查找表
  - `kernel/camera/camera.h` — 相机相关的查找表（如光圈形状等）

## 实现细节 / 关键算法

### 坐标映射与钳制
所有维度的输入坐标均通过 `saturatef()` 钳制到 [0, 1] 范围，然后乘以 `(size - 1)` 映射到表索引空间。这确保了查找始终在有效范围内进行，不会发生越界访问。

### 分层插值策略
高维查找表通过递归调用低维函数实现：
- 3D 读取在 z 维度上分解为两次 2D 读取
- 2D 读取在 y 维度上分解为两次 1D 读取
- 1D 读取在相邻索引间进行基本的线性插值

### 提前返回优化
每个维度的插值都检查插值因子是否恰好为 0（`t == 0.0f`），若是则跳过第二次数据读取和插值运算，直接返回已读取的值。这在坐标恰好对齐网格点时可节省一半的内存访问。

### 数据存储
查找表数据通过 `kernel_data_fetch(lookup_table, index + offset)` 从全局查找表数组中读取。`offset` 参数允许多个不同用途的查找表共享同一个全局数组，各自通过不同偏移量定位。

## 关联文件
- `src/kernel/globals.h` — 全局数据定义
- `src/kernel/closure/bsdf_microfacet.h` — GGX 微面元 BSDF 使用查找表加速能量计算
- `src/kernel/closure/bsdf_sheen.h` — Sheen BSDF 使用查找表
- `src/kernel/camera/camera.h` — 相机模块使用查找表
- `src/scene/tables.cpp` — 主机端查找表数据的生成和上传
