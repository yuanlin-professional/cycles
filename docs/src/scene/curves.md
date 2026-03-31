# curves.h / curves.cpp — 曲线工具函数与粒子曲线数据

## 概述

`curves.h` 定义了 `ParticleCurveData` 类，用于存储从 Blender 粒子系统导出的曲线数据的中间表示。`curves.cpp` 实现了 `curvebounds()` 函数，用于精确计算 Catmull-Rom 样条曲线在某一维度上的包围区间。这两个文件为毛发系统提供底层数据结构和数学工具。

## 类与结构体

### ParticleCurveData

- **功能**: 从 Blender 粒子系统导出曲线数据时使用的中间存储结构
- **关键成员**:
  - `psys_firstcurve` (`array<int>`) — 每个粒子系统中首条曲线的索引
  - `psys_curvenum` (`array<int>`) — 每个粒子系统的曲线数量
  - `psys_shader` (`array<int>`) — 每个粒子系统的着色器索引
  - `psys_rootradius` / `psys_tipradius` (`array<float>`) — 根部/尖端半径
  - `psys_shape` (`array<float>`) — 半径沿曲线变化的形状参数
  - `psys_closetip` (`array<bool>`) — 是否闭合尖端（半径为0）
  - `curve_firstkey` (`array<int>`) — 每条曲线首个控制点的索引
  - `curve_keynum` (`array<int>`) — 每条曲线的控制点数量
  - `curve_length` (`array<float>`) — 每条曲线的长度
  - `curve_uv` (`array<float2>`) — 每条曲线的 UV 坐标
  - `curve_vcol` (`array<float4>`) — 每条曲线的顶点颜色
  - `curvekey_co` (`array<float3>`) — 所有控制点坐标
  - `curvekey_time` (`array<float>`) — 每个控制点沿曲线的归一化时间参数

## 核心函数

### curvebounds()

```cpp
void curvebounds(float *lower, float *upper, float3 *p, const int dim);
```

计算由 4 个控制点定义的 Catmull-Rom 样条在指定维度（dim=0/1/2 对应 x/y/z）上的精确最小值和最大值。

**算法**:
1. 计算 Catmull-Rom 权重系数 `curve_coef[0..3]`
2. 对三次多项式求导得到二次方程
3. 求判别式，解出极值点参数 `ta` 和 `tb`（限制在 [0,1] 范围内）
4. 取控制点 P1、P2 的值以及极值点处的值的最大/最小值作为边界

## 依赖关系

- **内部头文件**: `util/array.h`, `util/types.h`, `util/math.h`
- **被引用**: `scene/hair.cpp`（曲线包围盒计算使用 `curvebounds`）, `scene/object.cpp`, `scene/scene.cpp`

## 实现细节 / 关键算法

- **Catmull-Rom 权重**: 使用标准 Catmull-Rom 基函数，系数为：
  - `coef[0] = p1`（起始值）
  - `coef[1] = 0.5 * (-p0 + p2)`（一阶导数）
  - `coef[2] = 0.5 * (2*p0 - 5*p1 + 4*p2 - p3)`（二阶导数）
  - `coef[3] = 0.5 * (-p0 + 3*p1 - 3*p2 + p3)`（三阶导数）
- **极值求解**: 导数 `3*coef[3]*t^2 + 2*coef[2]*t + coef[1] = 0`，使用判别式 `discroot = coef[2]^2 - 3*coef[3]*coef[1]` 求根。

## 关联文件

- `scene/hair.h` / `scene/hair.cpp` — 毛发几何体，调用 `curvebounds()` 计算包围盒
- `scene/object.cpp` — 对象管理中引用曲线数据
