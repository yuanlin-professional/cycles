# image_vdb.h / image_vdb.cpp - OpenVDB/NanoVDB 体积数据加载器

## 概述

本文件实现了 OpenVDB 和 NanoVDB 格式的体积数据加载器 `VDBImageLoader`。它负责将 OpenVDB 稀疏体素网格转换为 NanoVDB 格式以供 GPU 渲染使用，支持 1/3/4 通道体积数据和多种精度（float、fp16、fpn）。同时提供从密集体素数据创建稀疏 OpenVDB 网格的功能。

## 类与结构体

### VDBImageLoader
- **继承**: `ImageLoader`
- **功能**: 加载 OpenVDB 网格并转换为 NanoVDB 格式用于渲染
- **关键成员**:
  - `grid_name` — 网格名称
  - `clipping` — 裁剪阈值（默认 0.001，低于此值的体素视为空）
  - `grid` — OpenVDB 网格指针（条件编译 `WITH_OPENVDB`）
  - `bbox` — 有效体素的包围盒
  - `nanogrid` — NanoVDB 网格句柄（条件编译 `WITH_NANOVDB`）
  - `precision` — NanoVDB 精度（默认 16 位）
- **关键方法**:
  - `load_metadata()` — 检查设备 NanoVDB 支持，将 OpenVDB 转换为已知类型，再转为 NanoVDB，设置数据类型和 3D 变换矩阵
  - `load_pixels()` — 将 NanoVDB 数据拷贝到设备缓冲区（对空网格填零）
  - `cleanup()` — 释放 OpenVDB 和 NanoVDB 内存
  - `is_vdb_loader()` — 返回 `true`，标识此加载器类型
  - `get_grid()` — 返回 OpenVDB 网格指针
  - `grid_from_dense_voxels()` — 从密集体素数组创建稀疏 OpenVDB 网格
- **虚方法**:
  - `load_grid()` — 可被派生类覆盖，实现自定义网格加载逻辑

## 核心函数

- `create_grid<GridType>()` — 模板函数（OpenVDB 条件编译），将密集体素数据转换为稀疏 OpenVDB 网格，设置 index-to-world 变换矩阵

## 依赖关系

- **内部头文件**: `scene/image.h`, `util/transform.h`
- **cpp 额外引用**: `util/log.h`, `util/nanovdb.h`, `util/openvdb.h`, `util/texture.h`; 条件编译: `openvdb/tools/Dense.h`
- **外部依赖**: OpenVDB（`openvdb::GridBase`, `openvdb::tools::Dense`, `openvdb::tools::copyFromDense`）, NanoVDB（`nanovdb::GridHandle`）
- **被引用**: `scene/image.cpp`（VDB 加载器引用和 `vdb_loader()` 方法）, `scene/volume.cpp`（体积渲染）

## 实现细节 / 关键算法

1. **OpenVDB 到 NanoVDB 转换**: `load_metadata()` 中先将 OpenVDB 网格转换为已知浮点类型（`openvdb_convert_to_known_type`），再通过 `openvdb_to_nanovdb()` 转换为 NanoVDB 格式，根据通道数和精度设置对应的 `IMAGE_DATA_TYPE_NANOVDB_*` 类型。
2. **3D 变换矩阵**: 从 OpenVDB 网格的 AffineMap 提取 4x4 变换矩阵，转换为 Cycles 的 `Transform` 格式（取逆得到 object->voxel_index 的变换）。
3. **空网格处理**: 若 OpenVDB 网格的活跃体素包围盒为空，设置类型为 `IMAGE_DATA_TYPE_NANOVDB_EMPTY`，仅分配 1 字节内存。
4. **密集到稀疏转换**: `grid_from_dense_voxels()` 使用 `openvdb::tools::copyFromDense` 将密集体素数组转为稀疏网格，应用 clipping 阈值。支持 FloatGrid（1 通道）、Vec3fGrid（3 通道）和 Vec4fGrid（4 通道）。
5. **精度选项**: 单通道支持三种精度：`precision=0`(FPN)、`precision=16`(FP16)、其他(Float)。

## 关联文件

- `scene/image.h/.cpp` — 图像管理系统（基类定义、VDB 加载器引用）
- `scene/volume.cpp` — 体积渲染（使用 VDBImageLoader 加载体积属性）
- `util/nanovdb.h` — NanoVDB 工具封装
- `util/openvdb.h` — OpenVDB 工具封装
