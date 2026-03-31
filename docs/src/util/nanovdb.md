# nanovdb.h / nanovdb.cpp - NanoVDB 体积数据转换工具

## 概述
本文件提供 NanoVDB 与 OpenVDB 之间的格式转换功能。NanoVDB 是 OpenVDB 的 GPU 友好紧凑表示，Cycles 在 GPU 渲染时使用 NanoVDB 格式进行体积纹理采样。主要实现两个方向的转换：OpenVDB 转 NanoVDB（用于将场景体积数据传输到设备），NanoVDB 转 OpenVDB MaskGrid（用于提取拓扑信息）。整个模块受 `WITH_NANOVDB` 编译宏保护。

## 类与结构体

### `NanoToOpenVDBMask<NanoBuildT>`（模板类，cpp 内部）
将 NanoVDB 网格的活跃体素拓扑转换为 OpenVDB `MaskGrid`：
- 模板参数 `NanoBuildT` 支持 `float`、`Fp16`、`FpN`、`Vec3f`、`Vec4f`
- 递归处理树结构的三级节点（Root -> Node2 -> Node1 -> Leaf）
- 使用 `nanovdb::forEach` 并行处理子节点

### `NanoToOpenVDBMaskOp`（结构体，cpp 内部）
函数对象封装，持有 `mask_grid` 结果，供 `nanovdb_grid_type_operation` 模板分发调用。

### `ToNanoOp`（结构体，cpp 内部）
OpenVDB 转 NanoVDB 的函数对象：
- `precision`: 精度控制（0=FpN, 16=Fp16, 其他=float32）
- `clipping`: 裁切阈值（小于此值的体素被停用）
- 支持 `FloatGrid`、`Vec3fGrid`、`Vec4fGrid`
- 速度网格（`Vec3fGrid`）启用 MinMax 统计

## 核心函数

| 函数 | 说明 |
|------|------|
| `openvdb::MaskGrid::Ptr nanovdb_to_openvdb_mask(const nanovdb::GridHandle<> &handle)` | NanoVDB 网格转 OpenVDB 掩码网格（仅拓扑） |
| `nanovdb::GridHandle<> openvdb_to_nanovdb(const openvdb::GridBase::ConstPtr &grid, int precision, float clipping)` | OpenVDB 网格转 NanoVDB，支持多精度和裁切 |

### 内部辅助函数
| 函数 | 说明 |
|------|------|
| `bool nanovdb_grid_type_operation(handle, op)` | NanoVDB 网格类型分发（float/Fp16/FpN/Vec3f/Vec4f） |

### NanoVDB 版本兼容性
代码通过 `NANOVDB_MAJOR_VERSION_NUMBER` / `NANOVDB_MINOR_VERSION_NUMBER` 宏处理 API 差异：
- NanoVDB >= 32.7（OpenVDB 12）：使用 `nanovdb::tools::createNanoGrid`
- NanoVDB >= 32.6（OpenVDB 11）：使用 `nanovdb::createNanoGrid`
- 更早版本（OpenVDB 10）：使用 `nanovdb::openToNanoVDB`

## 依赖关系
- **内部头文件**: `util/openvdb.h`, `util/log.h`
- **外部依赖**: `<openvdb/openvdb.h>`, `<nanovdb/NanoVDB.h>`, `<nanovdb/GridHandle.h>`, `<openvdb/tools/Activate.h>`
- **编译条件**: `WITH_NANOVDB`
- **被引用**: `scene/image_vdb.cpp`（体积纹理加载）

## 关联文件
- `util/openvdb.h` - OpenVDB 网格类型操作
- `util/texture.h` - NanoVDB 纹理数据类型枚举
- `scene/image_vdb.cpp` - VDB 图像加载器
