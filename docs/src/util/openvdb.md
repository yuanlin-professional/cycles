# openvdb.h / openvdb.cpp - OpenVDB 体积网格操作工具

## 概述
本文件提供 Cycles 渲染器中 OpenVDB 体积网格的类型管理和转换工具。定义了自定义的 `Vec4fGrid` 类型（OpenVDB 原生不支持），实现了将各种 OpenVDB 网格类型转换为 Cycles 可渲染的已知类型（`FloatGrid`、`Vec3fGrid`、`Vec4fGrid`），并提供了基于类型的模板分发机制。整个模块受 `WITH_OPENVDB` 编译宏保护。

## 类与结构体

### 自定义 OpenVDB 类型
在 `openvdb` 命名空间中扩展定义：
```cpp
using Vec4fTree = tree::Tree4<Vec4f, 5, 4, 3>::Type;
using Vec4fGrid = Grid<Vec4fTree>;
```
这是一个四分量浮点体积网格，OpenVDB 标准库中不提供此类型。

### `NumChannelsOp`（结构体，cpp 内部）
用于查询网格通道数的函数对象：
- `FloatGrid` -> 1 通道
- `Vec3fGrid` -> 3 通道
- `Vec4fGrid` -> 4 通道

## 核心函数

| 函数 | 说明 |
|------|------|
| `openvdb::GridBase::ConstPtr openvdb_convert_to_known_type(const openvdb::GridBase::ConstPtr &grid)` | 将任意 OpenVDB 网格类型转为 Cycles 可渲染的类型 |
| `bool openvdb_grid_type_operation(const openvdb::GridBase::ConstPtr &grid, OpType &&op)` | 模板分发函数，对已知网格类型调用操作对象 |
| `int openvdb_num_channels(const openvdb::GridBase::ConstPtr &grid)` | 获取网格通道数（1/3/4） |

### 类型转换对照表（`openvdb_convert_to_known_type`）
| 源类型 | 目标类型 | 说明 |
|--------|----------|------|
| `FloatGrid` | 直传 | 已是已知类型 |
| `Vec3fGrid` | 直传 | 已是已知类型 |
| `Vec4fGrid` | 直传 | 已是已知类型 |
| `BoolGrid` | `FloatGrid` | 隐式转换 |
| `DoubleGrid` | `FloatGrid` | 精度截断 |
| `Int32Grid` | `FloatGrid` | 整型转浮点 |
| `Int64Grid` | `FloatGrid` | 整型转浮点 |
| `Vec3IGrid` | `Vec3fGrid` | 整型向量转浮点向量 |
| `Vec3dGrid` | `Vec3fGrid` | 双精度转单精度 |
| 其他 | `nullptr` | 不支持的类型 |

### `openvdb_grid_type_operation` 模板分发
通过 `op.template operator()<GridType, FloatDataType, channels>()` 调用，支持三种网格类型。用于 NanoVDB 转换、通道数查询等需要类型特化的操作。

## 依赖关系
- **外部依赖**: `<openvdb/openvdb.h>`, `<openvdb/tools/Activate.h>`, `<openvdb/tools/Dense.h>`
- **编译条件**: `WITH_OPENVDB`
- **被引用**: `util/nanovdb.cpp`, `scene/image_vdb.cpp`

## 关联文件
- `util/nanovdb.h` - NanoVDB 转换（依赖本文件的类型分发）
- `util/texture.h` - 定义 NanoVDB 纹理数据类型
- `scene/image_vdb.cpp` - VDB 体积图像加载器
