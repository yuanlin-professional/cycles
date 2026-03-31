# motion_triangle_shader.h - 运动模糊三角形着色数据设置

## 概述
本文件实现了运动模糊(motion blur)三角形在交点处的着色数据(ShaderData)初始化。将运动三角形特有的着色设置逻辑(插值位置、面法线、平滑法线、偏微分等)集中到一个函数中，便于共享插值位置和法线的计算结果，避免重复计算。

## 核心函数

### motion_triangle_shader_setup()
- **签名**: `ccl_device_noinline void motion_triangle_shader_setup(KernelGlobals kg, ShaderData *sd)`
- **功能**: 为运动三角形的交点设置完整的着色数据。执行以下步骤：
  1. **获取着色器**: 从 `tri_shader` 数组读取图元的着色器 ID
  2. **计算运动信息**: 调用 `motion_triangle_compute_info()` 获取时间步、插值因子和顶点索引
  3. **插值顶点位置**: 调用 `motion_triangle_vertices()` 获取当前时刻的三角形顶点
  4. **计算精确位置**: 调用 `motion_triangle_point_from_uv()` 使用重心坐标计算交点位置 sd->P
  5. **计算面法线**: 通过叉积计算几何法线 Ng，考虑负缩放(negative scale)情况下的法线翻转
  6. **计算 dPdu/dPdv**: 计算位置关于 uv 的偏微分 (v1-v0) 和 (v2-v0)
  7. **计算平滑法线**: 若着色器设置了 `SHADER_SMOOTH_NORMAL`，则调用 `motion_triangle_smooth_normal()` 在顶点法线间插值

## 依赖关系
- **内部头文件**: `kernel/globals.h`, `kernel/types.h`, `kernel/geom/motion_triangle.h`, `kernel/geom/motion_triangle_intersect.h`
- **被引用**: `geom/shader_data.h`

## 实现细节 / 关键算法

### 计算优化
函数被标记为 `ccl_device_noinline`(不内联)，因为运动三角形的着色设置涉及大量计算（顶点插值、法线计算等），内联会增加调用点的代码膨胀。

### 法线计算
- **面法线**: `Ng = normalize(cross(v1 - v0, v2 - v0))`，负缩放时反转叉积顺序
- **平滑法线**: 需要单独获取插值后的顶点法线，通过 `motion_triangle_smooth_normal()` 在三个顶点法线间基于重心坐标插值
- Ng 同时赋给 `sd->Ng` 和 `sd->N`，平滑法线仅覆盖 `sd->N`

### 偏微分
`dPdu = v1 - v0` 和 `dPdv = v2 - v0` 与静态三角形计算方式一致，受 `__DPDU__` 预处理宏保护。

## 关联文件
- `kernel/geom/motion_triangle.h` - 运动三角形顶点和法线插值
- `kernel/geom/motion_triangle_intersect.h` - 提供 `motion_triangle_point_from_uv()`
- `kernel/geom/shader_data.h` - 着色数据初始化主入口调用本函数
