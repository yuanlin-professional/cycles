# types_int2.h - 二维整数向量类型定义

## 概述

`types_int2.h` 定义了二维整数向量结构体 `int2`，包含 `x`、`y` 两个 `int` 分量。该类型主要用于二维像素坐标、图像尺寸、Tile 索引等整数空间的二维量表示。在 GPU 原生向量类型可用时使用平台内置定义。

## 类与结构体

### `struct int2`

仅在非原生向量类型环境下定义：

| 成员 | 类型 | 说明 |
|------|------|------|
| `x` | `int` | 第一分量 |
| `y` | `int` | 第二分量 |

**运算符重载（仅 CPU）**：
- `int operator[](int i) const` — 按索引只读访问（带断言边界检查 0-1）
- `int &operator[](int i)` — 按索引读写访问

## 核心函数/运算符重载

| 函数签名 | 功能说明 |
|----------|----------|
| `int2 make_int2(int x, int y)` | 从两个标量构造 `int2` |
| `int2 make_int2(int i)` | 广播单一标量到所有分量 |

## 依赖关系

- **内部头文件**:
  - `util/types_base.h`
- **被引用**: `util/types_float2.h`、`util/types.h`、`util/math_int2.h`、`integrator/tile.h`、`integrator/work_tile_scheduler.h`

## 关联文件

- `src/util/math_int2.h` — `int2` 的数学运算
- `src/util/types_float2.h` — 对应的浮点二维向量（引用本文件进行类型转换）
