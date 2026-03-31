# math.h - 数学模块统一入口头文件

## 概述

`math.h` 是 Cycles 渲染器数学工具库的顶层聚合头文件。它不包含任何独立的函数实现，而是通过 `#include` 将所有子模块统一导出，使调用方只需引入一个头文件即可使用完整的标量、向量、矩阵及矩形运算功能。

该文件采用 `IWYU pragma: export` 指令，确保包含分析工具 (Include-What-You-Use) 将子模块的符号视为由本文件直接导出。

## 核心函数

本文件不定义任何函数，仅负责聚合导出。

## 依赖关系

- **内部头文件**:
  - `util/types.h` — 基础类型定义
  - `util/math_base.h` — 标量数学基础运算
  - `util/math_int2.h` — int2 向量运算
  - `util/math_int3.h` — int3 向量运算
  - `util/math_int4.h` — int4 向量运算
  - `util/math_int8.h` — vint8 向量运算
  - `util/math_float2.h` — float2 向量运算
  - `util/math_float4.h` — float4 向量运算
  - `util/math_float8.h` — vfloat8 向量运算
  - `util/math_float3.h` — float3 向量运算（依赖 float4，故排在其后）
  - `util/math_dual.h` — 对偶数（自动微分）运算
  - `util/rect.h` — 矩形区域运算
- **被引用**: `util/transform.h`, `util/boundbox.h`, `util/hash.h`, `util/half.h`, `util/color.h` 以及大量 kernel、scene、test 文件

## 实现细节 / 关键算法

无独立实现。引入顺序经过精心安排：`float3` 在 `float4` 之后引入，因为其 SSE 实现依赖 `float4` 的 shuffle 等操作；`math_dual.h` 最后引入，因为对偶数运算建立在所有基础向量类型之上。

## 关联文件

- `util/types.h` — 与之配套的类型聚合头文件
- `util/transform.h` — 变换矩阵运算，是 math 的上层使用者
