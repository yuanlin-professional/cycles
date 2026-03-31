# util_transform_test.cpp - 变换矩阵运动分解测试

## 概述

此文件测试 Cycles 工具库中变换矩阵的运动分解（Motion Decompose）功能。运动分解将 4x4 变换矩阵分解为平移、旋转和缩放分量，用于运动模糊（Motion Blur）效果的插值计算。测试主要关注退化（degenerated）变换矩阵的处理，确保分解结果始终是有限的、安全的。

## 测试用例

### TEST(transform_motion_decompose, Degenerated)
- **功能**: 验证 `transform_motion_decompose` 函数对退化变换矩阵（如零缩放矩阵）的处理，确保不会产生 NaN 或无穷大值。
- **验证场景**:
  - **单个退化矩阵**: 对缩放为 `(0, 0, 0)` 的变换矩阵进行分解，验证结果 `DecomposedTransform` 中所有值均为有限数（`transform_decomposed_isfinite_safe` 返回 `true`）
  - **从前一帧复制到当前帧**: 两帧运动序列中第二帧为退化矩阵，验证分解后第二帧的数据从第一帧（有效的旋转变换）复制而来，即 `decomp[1].x ≈ decomp[0].x`（误差 < 1e-6）
  - **从下一帧复制到当前帧**: 两帧运动序列中第一帧为退化矩阵，验证分解后第一帧的数据从第二帧（有效的旋转变换）复制而来，即 `decomp[0].x ≈ decomp[1].x`（误差 < 1e-6）

## 依赖关系
- **被测源文件**: `util/transform.h`
- **测试框架**: Google Test (GTest)
- **辅助依赖**: `util/vector.h`

## 关联文件
- `src/util/transform.h` - 变换矩阵函数声明
- `src/util/transform.cpp` - 变换矩阵函数实现（含 `transform_motion_decompose`）
