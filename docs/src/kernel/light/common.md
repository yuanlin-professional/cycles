# common.h - 光源采样公共数据结构与工具函数

## 概述
本文件定义了 Cycles 渲染器光源子系统的核心数据结构 `LightSample` 和一组基础工具函数。`LightSample` 存储了光源采样的完整结果，是所有光源类型之间的统一接口。工具函数包括椭圆/矩形采样、面积到立体角的 PDF 转换，以及光源着色器可见性检查。

## 核心函数

### LightSample (结构体)
- **功能**: 存储光源采样结果的核心数据结构，包含以下字段：
  - `P`: 光源上的位置（远距光为方向）
  - `Ng`: 光源法线
  - `t`: 到光源的距离（远距光为 FLT_MAX）
  - `D`: 着色点到光源的方向
  - `u, v`: 图元上的参数化坐标
  - `pdf`: 选择光源和光源上点的综合 PDF
  - `pdf_selection`: 光源选择的 PDF
  - `eval_fac`: 强度乘子
  - `object`: 三角形/曲线光源的对象 ID
  - `prim`: 灯光的 lamp ID，三角形/曲线光源的图元 ID
  - `shader`: 着色器 ID
  - `group`: 光源组
  - `type`: 光源类型（LightType 枚举）
  - `emitter_id`: 发射体数组中的索引

### ellipse_sample()
- **签名**: `ccl_device_inline float3 ellipse_sample(const float3 ru, const float3 rv, const float2 rand)`
- **功能**: 在由两个轴向量 `ru` 和 `rv` 定义的椭圆上均匀采样一个点。内部使用 `sample_uniform_disk` 将均匀随机数映射到圆盘。

### rectangle_sample()
- **签名**: `ccl_device_inline float3 rectangle_sample(const float3 ru, const float3 rv, const float2 rand)`
- **功能**: 在由两个半轴向量 `ru` 和 `rv` 定义的矩形上均匀采样一个点。将 [0,1] 范围的随机数线性映射到 [-1,1]。

### disk_light_sample()
- **签名**: `ccl_device float3 disk_light_sample(const float3 n, const float2 rand)`
- **功能**: 在以法线 `n` 为朝向的单位圆盘上采样。先构建正交基然后调用 `ellipse_sample`。用于点光源和聚光灯的圆盘模式采样。

### light_pdf_area_to_solid_angle()
- **签名**: `ccl_device float light_pdf_area_to_solid_angle(const float3 Ng, const float3 I, const float t)`
- **功能**: 将面积度量下的 PDF 转换为立体角度量下的 PDF。公式为 `t^2 / cos(theta)`，其中 theta 是光线入射方向与光源法线的夹角。当 cos(theta) <= 0 时返回 0（光源背面不可见）。

### is_light_shader_visible_to_path()
- **签名**: `ccl_device_inline bool is_light_shader_visible_to_path(const int shader, const uint32_t path_flag)`
- **功能**: 检查光源着色器对当前路径类型是否可见。支持按路径类型（漫反射、光泽反射、透射、相机、散射）排除光源，用于实现光源的光线可见性控制。

## 依赖关系
- **内部头文件**: `kernel/types.h`, `kernel/sample/mapping.h`
- **被引用**: `kernel/light/area.h`, `kernel/light/background.h`, `kernel/light/distant.h`, `kernel/light/distribution.h`, `kernel/light/point.h`, `kernel/light/spot.h`, `kernel/light/tree.h`, `kernel/light/triangle.h`

## 实现细节 / 关键算法
1. **面积到立体角转换**: `light_pdf_area_to_solid_angle` 是光源采样中最核心的 PDF 转换工具。从面积度量 `dA` 到立体角度量 `d_omega` 的转换为 `d_omega = dA * cos(theta) / r^2`，因此 PDF 的转换因子为 `r^2 / cos(theta)`。
2. **着色器可见性**: 通过位运算检查 `SHADER_EXCLUDE_*` 标志与 `PATH_RAY_*` 路径类型的组合，实现高效的光源可见性过滤。这是 Blender 中"光线可见性"面板功能的内核实现。
3. **统一坐标约定**: `LightSample` 中的 `u, v` 坐标采用与 Embree 和 OptiX 一致的重心坐标表示法，确保跨后端的一致性。

## 关联文件
- `kernel/types.h` - 提供 `LightType` 枚举和其他基础类型定义
- `kernel/sample/mapping.h` - 提供 `sample_uniform_disk` 等采样映射函数
- 所有 `kernel/light/` 目录下的文件都依赖本文件
