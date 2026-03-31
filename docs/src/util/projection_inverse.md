# projection_inverse.h - 4x4 矩阵求逆实现

## 概述

`projection_inverse.h` 实现了 4x4 浮点矩阵的就地求逆算法。该实现源自 Industrial Light & Magic (ILM) 的开源代码，使用 Gauss-Jordan 消元法（含部分主元选取）。它被 `projection.h` 中的 `projection_inverse` 函数调用，用于计算投影矩阵的逆矩阵。

## 核心函数

| 函数 | 说明 |
|------|------|
| `projection_inverse_impl(R[4][4], M[4][4])` | 对矩阵 M 执行 Gauss-Jordan 消元，结果写入 R。M 被原地修改为单位矩阵。返回 false 表示矩阵奇异（不可逆） |

## 依赖关系

- **内部头文件**: `util/defines.h`（`UNLIKELY` 宏）
- **被引用**: `util/projection.h`（通过 `#include` 在非 Metal 平台引入）

## 实现细节 / 关键算法

### Gauss-Jordan 消元法

算法分为两个阶段：

1. **前向消元（Forward Elimination）**:
   - 对每一列（i=0..3），在该列的对角元素及其下方元素中寻找绝对值最大的元素作为主元（部分主元选取，Partial Pivoting）
   - 如果最大主元为 0，则矩阵奇异，返回 false
   - 如果主元不在对角位置，交换整行（同时交换 M 和 R 的对应行）
   - 用主元行消去该列下方所有行的对应元素：`M[j][k] -= f * M[i][k]`，同时对 R 执行相同操作

2. **回代（Backward Substitution）**:
   - 从最后一行开始，将对角元素归一化为 1（除以 `M[i][i]`）
   - 用当前行消去该列上方所有行的对应元素
   - 如果任一对角元素为 0，返回 false

### 数值特性

- **部分主元选取**: 选择每列中绝对值最大的元素作为主元，减少除法中的舍入误差传播
- **奇异性检测**: 在两处检查零主元——前向消元时最大主元为零，以及回代时对角元素为零
- **精度**: 单精度浮点，适用于渲染中的投影矩阵（通常条件数不高）
- **Metal 排除**: Metal Shading Language 平台不包含此文件，可能使用 Metal 内建的矩阵求逆

## 关联文件

- `util/projection.h` — 调用者，将 `ProjectionTransform` 拷贝到 `float[4][4]` 后调用本函数
- `util/transform.h` — 3x4 仿射变换有独立的求逆实现（利用旋转矩阵正交性优化）
