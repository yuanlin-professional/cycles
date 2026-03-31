# hair.h / hair.cpp — 毛发几何体

## 概述

`hair.h` 和 `hair.cpp` 实现了 Cycles 的毛发几何类 `Hair`，继承自 `Geometry`。毛发由一系列曲线组成，每条曲线由多个控制点（key）定义，使用 Catmull-Rom 样条进行插值。支持带状（Ribbon）和粗线（Thick）两种曲线形状，以及运动模糊和阴影透明度计算。

## 类与结构体

### Hair

- **继承**: `Geometry`
- **功能**: 毛发/曲线几何体，管理曲线控制点、半径和着色器数据
- **关键成员**:
  - `curve_keys` (`array<float3>`) — 所有曲线控制点坐标（全局数组）
  - `curve_radius` (`array<float>`) — 每个控制点的半径
  - `curve_first_key` (`array<int>`) — 每条曲线的首个控制点索引
  - `curve_shader` (`array<int>`) — 每条曲线的着色器索引
  - `curve_key_offset` (`size_t`) — 控制点在全局设备数组中的偏移
  - `curve_segment_offset` (`size_t`) — 曲线段在全局设备数组中的偏移
  - `curve_shape` (`CurveShapeType`) — 曲线形状：`CURVE_RIBBON`（带状）、`CURVE_THICK`（粗线 Catmull-Rom）、`CURVE_THICK_LINEAR`（粗线线性）
- **关键方法**:
  - `resize_curves()` / `reserve_curves()` — 调整/预分配曲线数据
  - `add_curve_key()` — 添加控制点
  - `add_curve()` — 添加一条新曲线
  - `copy_center_to_motion_step()` — 复制当前位置到运动模糊步
  - `compute_bounds()` — 使用 TBB 并行计算包围盒
  - `apply_transform()` — 应用变换（坐标和半径按均匀缩放因子缩放）
  - `get_uv_tiles()` — 获取 UDIM 瓦片
  - `pack_curves()` — 将曲线数据打包为设备格式（`KernelCurve` + `KernelCurveSegment`）
  - `primitive_type()` — 根据形状和运动模糊返回对应图元类型
  - `need_shadow_transparency()` — 检查是否需要阴影透明度
  - `update_shadow_transparency()` — 在设备上求值阴影透明度着色器
  - `num_keys()` / `num_curves()` / `num_segments()` — 数量查询
  - `get_curve()` — 获取第 i 条曲线结构体
  - `is_traceable()` — 是否有可追踪的曲线段

### Hair::Curve

- **功能**: 单条曲线的辅助结构
- **关键成员**: `first_key` — 首个控制点索引, `num_keys` — 控制点数量
- **关键方法**:
  - `num_segments()` — 返回 `num_keys - 1`
  - `bounds_grow()` — 多个重载，使用 Catmull-Rom 曲线的精确边界进行包围盒扩展
  - `motion_keys()` — 在运动步之间线性插值控制点
  - `cardinal_motion_keys()` — Cardinal 样条运动模糊插值（4 点）
  - `keys_for_step()` / `cardinal_keys_for_step()` — 获取特定运动步的控制点

## 核心函数

### 包围盒计算

`Hair::compute_bounds()` 使用 TBB `parallel_reduce` 并行遍历所有曲线的段，调用 `curvebounds()`（定义在 `curves.cpp`）精确计算 Catmull-Rom 样条的轴对齐包围盒。运动模糊步的控制点也被纳入包围盒。

### 阴影透明度

`update_shadow_transparency()` 通过 `ShaderEval` 在设备上对每个控制点求值 `SHADER_EVAL_CURVE_SHADOW_TRANSPARENCY`，将结果存储在 `ATTR_STD_SHADOW_TRANSPARENCY` 属性中。如果所有控制点完全不透明，则移除该属性以节省内存。

### 数据打包

`pack_curves()` 将控制点打包为 `float4`（xyz=坐标, w=半径），每条曲线生成 `KernelCurve` 结构（包含着色器ID、首控制点偏移、控制点数、图元类型），每个段生成 `KernelCurveSegment`。

## 依赖关系

- **内部头文件**: `scene/geometry.h`
- **被引用**: `scene/object.cpp`, `scene/geometry.cpp`, `scene/geometry_mesh.cpp`, `scene/geometry_attributes.cpp`, `scene/attribute.cpp`, `scene/scene.cpp`

## 实现细节 / 关键算法

- **Catmull-Rom 包围盒**: `bounds_grow()` 使用 4 个控制点（前一、当前、下一、后一）计算样条系数，通过求导找极值点来精确计算每个维度的最大/最小值。
- **均匀缩放半径**: `apply_transform()` 从变换矩阵的三列向量计算 `scalar = pow(|det|, 1/3)`，仅在均匀缩放下正确。
- **运动模糊存储**: 中心步的控制点存储在 `curve_keys` 中，其余步存储在 `ATTR_STD_MOTION_VERTEX_POSITION` 属性（`float4` 格式）中。
- **图元类型**: 6 种组合 = {RIBBON, THICK, THICK_LINEAR} x {静态, 运动}

## 关联文件

- `scene/curves.h` / `scene/curves.cpp` — `ParticleCurveData` 和 `curvebounds()` 函数
- `scene/geometry.h` — 基类
- `scene/object.cpp` — 对象引用毛发几何体
