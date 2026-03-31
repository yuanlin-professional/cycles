# patch.h / patch.cpp - 细分曲面面片定义与求值

## 概述

本文件定义了细分曲面的面片（Patch）抽象基类及其两种具体实现：线性四边形面片（LinearQuadPatch）和双三次贝塞尔面片（BicubicPatch）。面片是细分曲面系统的基本几何单元，提供在参数域 (u, v) 上求值位置、偏导数和法线的能力。这些面片类型在 DiagSplit 分割和 EdgeDice 切割过程中被调用以获取精确的几何信息。

## 类与结构体

### Patch

- **功能**: 面片抽象基类，定义所有面片类型的通用接口
- **关键成员**:
  - `patch_index` — 面片索引（用于 PTex 和 OpenSubdiv 面片查找），默认 0
  - `shader` — 着色器索引，默认 0
  - `smooth` — 是否使用平滑着色，默认 true
  - `from_ngon` — 是否来源于 N-gon 面（非四边形），默认 false
- **关键方法**:
  - `eval(P, dPdu, dPdv, N, u, v)` — 纯虚函数，在参数坐标 (u, v) 处求值位置 P、U 方向偏导 dPdu、V 方向偏导 dPdv、法线 N

### LinearQuadPatch

- **继承**: `Patch`（final）
- **功能**: 线性四边形面片，使用4个控制点进行双线性插值
- **关键成员**:
  - `hull[4]` — 4个角点坐标（`float3`数组）
- **关键方法**:
  - `eval(...)` — 双线性插值求值：先沿 u 方向线性插值两对顶点，再沿 v 方向插值结果；法线通过偏导数叉积计算
  - `bound()` — 计算4个控制点的轴对齐包围盒

### BicubicPatch

- **继承**: `Patch`（final）
- **功能**: 双三次贝塞尔面片，使用 4x4 = 16 个控制点
- **关键成员**:
  - `hull[16]` — 16个控制点坐标（`float3`数组）
- **关键方法**:
  - `eval(...)` — 使用 De Casteljau 算法进行双三次贝塞尔求值，法线通过偏导数叉积计算
  - `bound()` — 计算16个控制点的轴对齐包围盒

## 枚举与常量

无。

## 核心函数

### decasteljau_cubic()
- **签名**: `static void decasteljau_cubic(float3 *P, float3 *dt, const float t, const float3 cp[4])`
- **功能**: De Casteljau 三次贝塞尔曲线求值。给定4个控制点和参数 t，计算曲线上的点 P 以及切线方向 dt。通过三级线性插值递推实现。

### decasteljau_bicubic()
- **签名**: `static void decasteljau_bicubic(float3 *P, float3 *du, float3 *dv, const float3 cp[16], float u, const float v)`
- **功能**: De Casteljau 双三次贝塞尔曲面求值。先沿 u 方向对4行各4个控制点分别求值得到4个中间点和4个切线，再沿 v 方向对这4个中间点求值得到最终位置和 v 方向偏导数；u 方向偏导数则沿 v 方向对切线数组求值得到。

### LinearQuadPatch::eval()
- **签名**: `void eval(float3 *P, float3 *dPdu, float3 *dPdv, float3 *N, const float u, float v) const`
- **功能**: 双线性插值求值。P = lerp(lerp(hull[0], hull[1], u), lerp(hull[2], hull[3], u), v)。偏导数分别为 u 和 v 方向的差分插值，法线为偏导数叉积的归一化。

### BicubicPatch::eval()
- **签名**: `void eval(float3 *P, float3 *dPdu, float3 *dPdv, float3 *N, const float u, const float v) const`
- **功能**: 调用 decasteljau_bicubic 进行双三次求值。若需要法线，先计算偏导数再叉积归一化。

## 依赖关系

- **内部头文件**: `util/boundbox.h`, `util/types.h`, `util/math.h`
- **外部库**: 无
- **被引用**: `subd/osd.h`, `subd/split.cpp`, `subd/dice.cpp`, `scene/mesh_subdivision.cpp`

## 实现细节 / 关键算法

1. **De Casteljau 算法**: 这是计算贝塞尔曲线/曲面上点的经典数值稳定算法。相比直接展开多项式，De Casteljau 算法通过逐级线性插值实现，数值稳定性更好。三次曲线需要3级插值，双三次曲面则是先对4行分别做三次求值，再对结果做一次三次求值。
2. **偏导数计算**: 对于三次贝塞尔曲线 C(t)，其导数 C'(t) = 3 * decasteljau_quadratic(差分控制点, t)。在 De Casteljau 中间过程中，最后两个中间点的差即为（未缩放的）切线方向。
3. **法线计算**: 法线 N = normalize(cross(dPdu, dPdv))，对于 LinearQuadPatch 和 BicubicPatch 均适用。
4. **包围盒计算**: 对于贝塞尔面片，控制点的凸包包含整个面片，因此对所有控制点取最小最大值即可得到保守的轴对齐包围盒。

## 关联文件

- `subd/osd.h` — OsdPatch 继承自 Patch，提供 OpenSubdiv 求值
- `subd/split.h` / `subd/split.cpp` — DiagSplit 使用 Patch::eval 计算边缘因子
- `subd/dice.h` / `subd/dice.cpp` — EdgeDice 使用 Patch::eval 设置顶点位置
- `scene/mesh_subdivision.cpp` — 创建 LinearQuadPatch/BicubicPatch 面片数组
