# kernel_camera_projection_test.cpp - 相机投影模型测试

## 概述

此文件测试 Cycles 渲染器内核中的相机投影函数，主要覆盖鱼眼镜头多项式模型（Fisheye Lens Polynomial）以及多种全景投影模型的正向和反向映射的正确性。测试通过往返验证（round-trip）确保传感器坐标到方向向量、方向向量回传感器坐标的转换精度。

## 测试用例

### TEST(KernelCamera, FisheyeLensPolynomialRoundtrip)
- **功能**: 验证鱼眼镜头多项式投影模型的往返一致性。对多组镜头参数（等距、立体投影、直线型）和多个传感器点坐标，检查传感器坐标 -> 方向向量 -> 传感器坐标的往返误差是否在容差范围内（1e-6）。
- **参数组合**: 3 种镜头系数 x 5 种 k0 值 x 5 种 k2 值 x 6 个传感器点

### TEST(KernelCamera, FisheyeLensPolynomialToDirectionSymmetry)
- **功能**: 验证等距鱼眼模型的对称性。对于以传感器中心为对称点的一对坐标，其生成的方向向量应满足特定的对称关系（x 分量相等，y 和 z 分量取反）。
- **验证数据**: 10 组关于中心对称的偏移量

### TEST(KernelCamera, FisheyeLensPolynomialToDirection)
- **功能**: 使用手工计算的参考值验证等距鱼眼模型的投影函数。测试 30 度、45 度、60 度等多个角度的投影结果，并在多种缩放比例下验证正向投影和反向重投影的精度。
- **验证数据**: 17 组角度参考值 x 9 种缩放比例

### TYPED_TEST(PanoramaProjection, round_trip)
- **功能**: 使用类型化测试（Typed Test）验证多种全景投影模型的往返一致性。
- **覆盖的投影模型**:
  - `Spherical` - 球面投影
  - `Equirectangular` - 等距柱状投影
  - `FisheyeEquidistant` - 等距鱼眼投影
  - `FisheyeEquisolid` - 等面积鱼眼投影
  - `MirrorBall` - 镜面球投影
  - `EquiangularCubemapFace` - 等角立方体面投影
- **参数组合**: 3 种传感器尺寸 x 6 种视场角 x 10 个传感器点

## 辅助结构体

| 结构体 | 说明 |
|--------|------|
| `CommonValues` | 测试基类，定义重投影误差阈值和无效投影处理策略 |
| `Spherical` | 球面投影适配器 |
| `Equirectangular` | 等距柱状投影适配器 |
| `FisheyeEquidistant` | 等距鱼眼投影适配器 |
| `FisheyeEquisolid` | 等面积鱼眼投影适配器（跳过无效反投影） |
| `MirrorBall` | 镜面球投影适配器 |
| `EquiangularCubemapFace` | 等角立方体面投影适配器 |

## 依赖关系
- **被测源文件**: `kernel/camera/projection.h`
- **测试框架**: Google Test (GTest)，使用 `TYPED_TEST_SUITE` 特性
- **辅助依赖**: `util/math.h`, `util/types.h`

## 关联文件
- `src/kernel/camera/projection.h` - 相机投影函数定义
