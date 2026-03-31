# rect.h - 整数矩形区域运算

## 概述

`rect.h` 提供基于 `int4` 的 2D 整数矩形区域操作工具。矩形以 `int4(x0, y0, x1, y1)` 格式存储，其中 (x0, y0) 为左下角，(x1, y1) 为右上角。这些工具主要用于 Cycles 的图块(tile)渲染系统和降噪器中的像素区域管理。

## 核心函数

| 函数 | 说明 |
|------|------|
| `rect_from_shape(x0, y0, w, h)` | 从起点和宽高创建矩形 |
| `rect_expand(rect, d)` | 向外扩展 d 像素 |
| `rect_clip(a, b)` | 求两矩形的交集 |
| `rect_is_valid(rect)` | 矩形是否有效（宽高 > 0） |
| `rect_size(rect)` | 矩形像素数（宽 * 高） |
| `coord_to_local_index(rect, x, y)` | 像素坐标转为矩形内的行优先局部索引 |
| `local_index_to_coord(rect, idx, *x, *y)` | 局部索引转回像素坐标，返回是否在矩形内 |

## 依赖关系

- **内部头文件**: `util/math_base.h`, `util/types_int4.h`
- **被引用**: 通过 `util/math.h` 间接被引用（是最后一个被 include 的子模块）

## 实现细节 / 关键算法

1. **`rect_clip`**: 使用逐分量 max(左下角坐标) 和 min(右上角坐标) 实现矩形交集。当结果无效（x0>=x1 或 y0>=y1）时，`rect_is_valid` 返回 false。

2. **行优先索引**: `coord_to_local_index` 计算 `(y - y0) * width + (x - x0)`，这与降噪器中缓冲区的内存布局一致。

3. **矩形以 int4 表示**: 利用 int4 的 SIMD 特性，矩形的创建、扩展等操作可以被编译器向量化。

## 关联文件

- `util/types_int4.h` — int4 类型定义
- `util/math_int4.h` — int4 的 min/max 运算
- `util/math.h` — 本文件在聚合头文件中最后被引入
