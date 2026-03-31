# differential.h - 光线微分计算工具

## 概述
本文件实现了光线微分（Ray Differentials）的计算逻辑，用于追踪光线在传播和交互过程中的展开（spread）信息。光线微分是纹理抗锯齿（Mip-mapping）、自适应采样等功能的基础。文件同时提供了完整微分和紧凑微分（Compact Differentials）两套实现——前者精确但占用更多内存，后者将微分近似为标量半径（类似锥形追踪），适用于 GPU 内存受限场景。算法基于 Homan Igehy 1999 年的论文 "Tracing Ray Differentials"。

## 类与结构体
本文件使用但不定义以下结构体（定义在 `kernel/types.h` 中）：

- **`differential3`** — 三维微分，包含 `dx`（float3）和 `dy`（float3），表示沿屏幕 x/y 方向的微分分量
- **`differential`** — 标量微分，包含 `dx`（float）和 `dy`（float），用于 UV 坐标等标量参数的微分

## 枚举与常量
本文件未定义枚举或常量。

## 核心函数

### differential_transfer()
- **签名**: `ccl_device void differential_transfer(ccl_private differential3 *surface_dP, const differential3 ray_dP, const float3 ray_D, const differential3 ray_dD, const float3 surface_Ng, const float ray_t)`
- **功能**: 计算光线在均匀介质中传播后，交点处的位置微分 dP。将光线的位置微分和方向微分沿着传播距离 `ray_t` 向前推进，并投影到交点的切平面（由几何法线 `surface_Ng` 定义）。

### differential_incoming()
- **签名**: `ccl_device void differential_incoming(ccl_private differential3 *dI, const differential3 dD)`
- **功能**: 根据光线方向微分 dD 计算入射方向微分 dI。由于入射方向与光线方向相反，直接取负即可。

### differential_dudv()
- **签名**: `ccl_device void differential_dudv(ccl_private differential *du, ccl_private differential *dv, float3 dPdu, float3 dPdv, differential3 dP, const float3 Ng)`
- **功能**: 根据位置微分 dP 和参数化导数 dPdu/dPdv，利用克莱姆法则（Cramer's Rule）求解 UV 坐标的屏幕空间微分 du/dv。这些微分值主要用于纹理过滤和任意网格属性的微分计算。

### differential_zero()
- **签名**: `ccl_device differential differential_zero()`
- **功能**: 返回零初始化的标量微分结构体。

### differential3_zero()
- **签名**: `ccl_device differential3 differential3_zero()`
- **功能**: 返回零初始化的三维微分结构体。

### differential_zero_compact()
- **签名**: `ccl_device_forceinline float differential_zero_compact()`
- **功能**: 返回零值紧凑微分（标量 0.0f）。

### differential_make_compact() (三个重载)
- **签名**: `ccl_device_forceinline float differential_make_compact(const float dD)` / `(const differential3 dD)` / `(const dual3 D)`
- **功能**: 将完整微分压缩为标量半径。对于 `float` 输入直接返回；对于 `differential3` 和 `dual3` 输入，取 dx 和 dy 长度的平均值作为近似半径。

### differential_incoming_compact()
- **签名**: `ccl_device_forceinline float differential_incoming_compact(const float dD)`
- **功能**: 紧凑微分版本的入射方向微分计算。由于标量半径不区分方向，直接返回原值。

### differential_transfer_compact()
- **签名**: `ccl_device_forceinline float differential_transfer_compact(const float ray_dP, const float3 ray_D, const float ray_dD, const float ray_t)`
- **功能**: 紧凑微分版本的光线传播。沿传播距离线性扩展微分半径：`ray_dP + ray_t * ray_dD`。

### differential_from_compact()
- **签名**: `ccl_device_forceinline differential3 differential_from_compact(const float3 D, const float dD)`
- **功能**: 将紧凑标量微分恢复为完整的 `differential3`。以光线方向 D 为基础构建正交坐标系，将标量半径分配到两个正交方向。

### differential_dudv_compact()
- **签名**: `ccl_device void differential_dudv_compact(ccl_private differential *du, ccl_private differential *dv, const float3 dPdu, const float3 dPdv, const float dP, const float3 Ng)`
- **功能**: 紧凑微分版本的 UV 微分计算。先通过 `differential_from_compact()` 恢复为完整微分，再调用 `differential_dudv()` 计算。

## 依赖关系
- **内部头文件**:
  - `kernel/types.h` — 提供 `differential`、`differential3` 结构体定义及 `dual3` 类型
- **被引用**:
  - `kernel/svm/wireframe.h`、`kernel/svm/attribute.h`、`kernel/svm/bump.h`、`kernel/svm/displace.h` — 着色器虚拟机（SVM）节点
  - `kernel/osl/osl.h`、`kernel/osl/services_gpu.h`、`kernel/osl/closures.cpp` — OSL 着色器系统
  - `kernel/integrator/subsurface_disk.h`、`kernel/integrator/state_util.h` — 积分器
  - `kernel/geom/shader_data.h` — 着色器数据初始化
  - `kernel/camera/camera.h` — 相机光线生成

## 实现细节 / 关键算法

### 光线微分传播
`differential_transfer()` 实现了光线微分在均匀介质中的传播公式。对于传播距离 t 的光线：
- 首先计算传播后的总位移：`tmpx = ray_dP.dx + t * ray_dD.dx`
- 然后投影到交点切平面：`surface_dP.dx = tmpx - dot(tmpx, Ng) * (D / dot(D, Ng))`

### UV 微分的克莱姆法则求解
`differential_dudv()` 需要求解 2x2 线性方程组。为提高数值稳定性，先将 3D 问题投影到 2D——选择法线绝对值最大的分量对应的平面进行投影（即丢弃变化最小的维度），然后用克莱姆法则求解。

### 紧凑微分（锥形追踪）
紧凑微分模式将完整的 `differential3`（6 个浮点数）压缩为单个标量半径（1 个浮点数），本质上是将光线的微分椭圆近似为圆锥。这在 GPU 上可显著减少内存占用和访问开销，代价是损失微分的方向性信息。

## 关联文件
- `src/kernel/types.h` — `differential`、`differential3`、`dual3` 类型定义
- `src/kernel/camera/camera.h` — 相机光线的初始微分生成
- `src/kernel/geom/shader_data.h` — 着色点上的微分初始化
- `src/kernel/svm/bump.h` — 凹凸贴图中使用微分计算扰动
