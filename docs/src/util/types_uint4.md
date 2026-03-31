# types_uint4.h - 四维无符号整数向量类型定义

## 概述

`types_uint4.h` 定义了四维无符号整数向量结构体 `uint4`，包含 `x`、`y`、`z`、`w` 四个 `uint`（`unsigned int`）分量。该类型用于 BVH 节点数据打包、内核数据传递中需要四个无符号整数的场景。在 GPU 原生向量类型可用时使用平台内置定义。

## 类与结构体

### `struct uint4`

仅在非原生向量类型环境下定义：

| 成员 | 类型 | 说明 |
|------|------|------|
| `x` | `uint` | 第一分量 |
| `y` | `uint` | 第二分量 |
| `z` | `uint` | 第三分量 |
| `w` | `uint` | 第四分量 |

**运算符重载（仅 CPU）**：
- `uint operator[](uint i) const` — 按索引只读访问（断言检查上界为 3）
- `uint &operator[](uint i)` — 按索引读写访问

## 核心函数/运算符重载

| 函数签名 | 功能说明 |
|----------|----------|
| `uint4 make_uint4(uint x, uint y, uint z, uint w)` | 从四个无符号整数构造 `uint4` |

## 依赖关系

- **内部头文件**:
  - `util/types_base.h`
- **被引用**: `util/types.h`

## 关联文件

- `src/util/types_base.h` — `uint` 类型别名定义
- `src/util/types.h` — 类型聚合头文件
