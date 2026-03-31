# types_uchar2.h - 二维无符号字节向量类型定义

## 概述

`types_uchar2.h` 定义了二维无符号字节向量结构体 `uchar2`，包含 `x`、`y` 两个 `uchar`（`unsigned char`）分量。该类型用于需要低精度二分量数据的场景，如紧凑的像素数据对或小型索引。在 GPU 原生向量类型可用时使用平台内置定义。

## 类与结构体

### `struct uchar2`

仅在非原生向量类型环境下定义：

| 成员 | 类型 | 说明 |
|------|------|------|
| `x` | `uchar` | 第一分量（0-255） |
| `y` | `uchar` | 第二分量（0-255） |

**运算符重载（仅 CPU）**：
- `uchar operator[](int i) const` — 按索引只读访问（带断言边界检查 0-1）
- `uchar &operator[](int i)` — 按索引读写访问

## 核心函数/运算符重载

| 函数签名 | 功能说明 |
|----------|----------|
| `uchar2 make_uchar2(uchar x, uchar y)` | 从两个字节值构造 `uchar2` |

## 依赖关系

- **内部头文件**:
  - `util/types_base.h`
- **被引用**: `util/types.h`

## 关联文件

- `src/util/types_base.h` — `uchar` 类型别名定义
- `src/util/types.h` — 类型聚合头文件
