# util_boundbox_test.cpp - 包围盒变换测试

## 概述

此文件测试 Cycles 工具库中的轴对齐包围盒（AABB, Axis-Aligned Bounding Box）的变换功能。包围盒是光线追踪中 BVH 加速结构的基本构建单元，其变换操作的正确性直接影响渲染的正确性和性能。

## 测试用例

### TEST(BoundBox, transformed)
- **功能**: 验证 `BoundBox::transformed()` 方法在不同变换矩阵下的正确性。
- **验证场景**:
  - **平移变换**: 对包围盒 `[(-2,-3,-4), (3,4,5)]` 施加平移 `(1,2,3)`，验证变换后的包围盒为 `[(-1,-1,-1), (4,6,8)]`，误差容差 1e-6
  - **无效包围盒变换**: 对空/无效包围盒（`BoundBox::empty`）施加单位缩放变换，验证变换后的包围盒仍然无效（`valid()` 返回 `false`）

## 依赖关系
- **被测源文件**: `util/boundbox.h`
- **测试框架**: Google Test (GTest)
- **辅助依赖**: `util/transform.h`

## 关联文件
- `src/util/boundbox.h` - BoundBox 类定义
- `src/util/transform.h` - 变换矩阵函数定义
