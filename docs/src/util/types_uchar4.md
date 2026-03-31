# types_uchar4.h - 四维无符号字节向量类型定义

## 概述

`types_uchar4.h` 定义了四维无符号字节向量结构体 `uchar4`，包含 `x`、`y`、`z`、`w` 四个 `uchar`（`unsigned char`）分量。该类型广泛用于 RGBA 字节颜色表示、图像像素数据的紧凑存储。4 字节总大小使其可以直接与 32 位整数进行内存级别的转换。在 GPU 原生向量类型可用时使用平台内置定义。

## 类与结构体

### `struct uchar4`

仅在非原生向量类型环境下定义：

| 成员 | 类型 | 说明 |
|------|------|------|
| `x` | `uchar` | 第一分量 / R 通道（0-255） |
| `y` | `uchar` | 第二分量 / G 通道（0-255） |
| `z` | `uchar` | 第三分量 / B 通道（0-255） |
| `w` | `uchar` | 第四分量 / A 通道（0-255） |

**运算符重载（仅 CPU）**：
- `uchar operator[](int i) const` — 按索引只读访问（带断言边界检查 0-3）
- `uchar &operator[](int i)` — 按索引读写访问

## 核心函数/运算符重载

| 函数签名 | 功能说明 |
|----------|----------|
| `uchar4 make_uchar4(uchar x, uchar y, uchar z, uchar w)` | 从四个字节值构造 `uchar4` |
| `bool operator==(uchar4 a, uchar4 b)` | 逐分量相等比较 |

## 依赖关系

- **内部头文件**:
  - `util/types_base.h`
- **被引用**: `util/types.h`

## 关联文件

- `src/util/types_base.h` — `uchar` 类型别名定义
- `src/util/types.h` — 类型聚合头文件
