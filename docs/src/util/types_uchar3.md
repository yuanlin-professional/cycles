# types_uchar3.h - 三维无符号字节向量类型定义

## 概述

`types_uchar3.h` 定义了三维无符号字节向量结构体 `uchar3`，包含 `x`、`y`、`z` 三个 `uchar`（`unsigned char`）分量。该类型适用于紧凑的三通道字节数据存储，如 RGB 字节颜色。在 GPU 原生向量类型可用时使用平台内置定义。

## 类与结构体

### `struct uchar3`

仅在非原生向量类型环境下定义：

| 成员 | 类型 | 说明 |
|------|------|------|
| `x` | `uchar` | 第一分量（0-255） |
| `y` | `uchar` | 第二分量（0-255） |
| `z` | `uchar` | 第三分量（0-255） |

**运算符重载（仅 CPU）**：
- `uchar operator[](int i) const` — 按索引只读访问（带断言边界检查 0-2）
- `uchar &operator[](int i)` — 按索引读写访问

## 核心函数/运算符重载

| 函数签名 | 功能说明 |
|----------|----------|
| `uchar3 make_uchar3(uchar x, uchar y, uchar z)` | 从三个字节值构造 `uchar3` |

## 依赖关系

- **内部头文件**:
  - `util/types_base.h`
- **被引用**: `util/types.h`

## 关联文件

- `src/util/types_base.h` — `uchar` 类型别名定义
- `src/util/types.h` — 类型聚合头文件
