# types_uint2.h - 二维无符号整数向量类型定义

## 概述

`types_uint2.h` 定义了二维无符号整数向量结构体 `uint2`，包含 `x`、`y` 两个 `uint`（`unsigned int`）分量。该类型用于需要非负整数二维量的场景，如纹理尺寸、哈希值对、随机数种子等。在 GPU 原生向量类型可用时使用平台内置定义。

## 类与结构体

### `struct uint2`

仅在非原生向量类型环境下定义：

| 成员 | 类型 | 说明 |
|------|------|------|
| `x` | `uint` | 第一分量 |
| `y` | `uint` | 第二分量 |

**运算符重载（仅 CPU）**：
- `uint operator[](uint i) const` — 按索引只读访问（带断言边界检查）
- `uint &operator[](uint i)` — 按索引读写访问

注意：索引参数类型为 `uint` 而非 `int`。

## 核心函数/运算符重载

| 函数签名 | 功能说明 |
|----------|----------|
| `uint2 make_uint2(uint x, uint y)` | 从两个无符号整数构造 `uint2` |

## 依赖关系

- **内部头文件**:
  - `util/types_base.h`
- **被引用**: `util/types.h`

## 关联文件

- `src/util/types_base.h` — `uint` 类型别名定义
- `src/util/types.h` — 类型聚合头文件
