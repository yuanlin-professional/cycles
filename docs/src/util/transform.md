# transform.h / transform_avx2.cpp - 仿射变换矩阵运算与运动模糊插值

## 概述

`transform.h` 定义了 Cycles 渲染器的核心仿射变换系统，包括 4x3 变换矩阵（`Transform`）和分解变换（`DecomposedTransform`）数据结构，以及点/方向变换、矩阵构造、矩阵求逆、运动模糊插值等完整的变换运算函数集。该模块是整个渲染管线中几何变换的基础，支持 CPU（含 SSE/AVX2 优化）、CUDA、Metal、OptiX 等多平台后端。

`transform_avx2.cpp` 提供了矩阵求逆运算的 AVX2 优化编译单元，通过独立编译确保 AVX2 指令集仅在支持的硬件上执行。

## 核心函数/类

### 数据结构

#### `Transform`
- **存储格式**: 4x3 仿射变换矩阵，由 3 个 `float4` 行向量（`x`, `y`, `z`）组成，每行包含 3x3 旋转/缩放部分加 1 个平移分量
- **内存布局**: `[Rx Ry Rz Tx | Rx Ry Rz Ty | Rx Ry Rz Tz]`（行主序）
- **运算符**: 支持 `operator[]`（仅 CPU 端）、`operator*`（矩阵乘法）、`operator==` / `operator!=`（基于 `memcmp` 的比较，NaN 安全）

#### `DecomposedTransform`
- **功能**: 变换的分解表示，用于运动模糊中的变换插值
- **存储格式**: 4 个 `float4`（`x`, `y`, `z`, `w`），紧凑地打包旋转四元数（4）、平移（3）和 3x3 缩放矩阵（9）共 16 个浮点数

### 点与方向变换

#### `transform_point(t, a)`
- **功能**: 将点 `a` 通过变换矩阵 `t` 进行仿射变换（含平移）
- **重载**: 支持 `float3` 和 `dual3`（用于微分光线的偏导数变换）
- **优化**: SSE2 路径使用矩阵转置 + `madd` 指令链；Metal 路径使用内建 `float3x3` 乘法

#### `transform_direction(t, a)`
- **功能**: 将方向向量 `a` 通过变换矩阵 `t` 的 3x3 部分进行变换（不含平移）

#### `transform_direction_transposed(t, a)`
- **功能**: 使用变换矩阵的转置 3x3 部分变换方向向量，等价于逆转置变换（用于法线变换）

### 矩阵构造函数

#### `make_transform(...)`
- **重载 1**: 接受 12 个标量参数，按行构造 4x3 矩阵
- **重载 2**: 接受 3 个 `float3` 参数，构造仅包含 3x3 旋转部分的变换（平移为零）

#### `euler_to_transform(euler)`
- **功能**: 将欧拉角（XYZ 顺序，弧度）直接转换为旋转矩阵，不经过中间四元数表示

#### `make_transform_frame(N)`
- **功能**: 从归一化法线 `N` 构造正交坐标系变换矩阵（TBN 矩阵）

#### 仅 CPU 端的构造函数
- `transform_translate(t)` — 平移变换
- `transform_scale(s)` — 缩放变换
- `transform_rotate(angle, axis)` — 绕任意轴旋转变换（Rodrigues 公式）
- `transform_euler(euler)` — 欧拉角旋转（通过矩阵乘法组合 Z-Y-X 旋转）
- `transform_identity()` — 单位矩阵
- `transform_empty()` — 全零矩阵
- `transform_zero()` — 全零矩阵（GPU 可用版本）

### 矩阵求逆

#### `transform_inverse(tfm)`
- **功能**: 计算 4x3 仿射变换矩阵的逆矩阵
- **实现策略**:
  - CPU 端优先调用 AVX2 优化路径 `transform_inverse_cpu_avx2()`
  - 回退到通用实现 `transform_inverse_impl()`
- **算法**: 基于伴随矩阵法，与 Embree 的实现逐位匹配，以确保实例化光线追踪的精度一致性

#### `transform_inverse_impl(tfm)`
- **功能**: 通用矩阵求逆实现
- **退化处理**: 当行列式为零时（如某轴缩放为 0），对对角线添加 `1e-8f` 微扰后重试

#### `transform_inverse_cross(a, b)` / `transform_inverse_dot(a, b)`
- **功能**: 与 Embree 逐位匹配的叉积和点积实现，使用 AVX2 FMA 指令（`_mm_fmsub_ps`）和 SSE4.2 点积指令（`_mm_dp_ps`）

### 运动模糊

#### `quat_interpolate(q1, q2, t)`
- **功能**: 四元数球面线性插值（Slerp），用于旋转插值
- **优化**: 当 `costheta > 0.9995f` 时退化为线性插值；GPU 光线追踪后端（OptiX/MetalRT）始终使用线性插值以匹配硬件行为

#### `transform_compose(tfm, decomp)`
- **功能**: 从分解表示（四元数旋转 + 平移 + 缩放矩阵）重新组合为 4x3 变换矩阵

#### `transform_motion_array_interpolate(tfm, motion, numsteps, time)`
- **功能**: 在分解变换数组中进行时间插值，用于运动模糊。旋转使用 Slerp，平移和缩放使用线性插值

### 辅助函数（仅 CPU 端）

- `transform_transposed_inverse(tfm)` — 计算转置逆矩阵（声明，实现在 `transform.cpp`）
- `transform_uniform_scale(tfm, scale)` — 检测是否为等比缩放
- `transform_negative_scale(tfm)` — 检测是否包含负缩放（镜像变换）
- `transform_clear_scale(tfm)` — 移除缩放，仅保留旋转和平移
- `transform_get_column(t, column)` / `transform_set_column(t, column, value)` — 按列访问矩阵
- `transform_equal_threshold(A, B, threshold)` — 带阈值的近似相等比较
- `transform_to_quat(tfm)` — 变换矩阵转四元数（声明）
- `transform_motion_decompose(decomp, motion, size)` — 运动变换分解（声明）
- `transform_from_viewplane(viewplane)` — 从视平面构造投影变换（声明）
- `transform_isfinite_safe(tfm)` / `transform_decomposed_isfinite_safe(decomp)` — 有限值安全检查

## transform_avx2.cpp

### `transform_inverse_cpu_avx2(tfm, itfm)`
- **签名**: `void transform_inverse_cpu_avx2(const Transform &tfm, Transform &itfm)`
- **功能**: AVX2 优化的矩阵求逆，独立编译单元确保以 `-mavx2` 标志编译
- **实现**: 内部直接调用 `transform_inverse_impl()`，但由于编译时启用了 AVX2，其中的 `transform_inverse_cross()` 和 `transform_inverse_dot()` 将使用 FMA/SSE4.2 指令路径

## 依赖关系

- **内部头文件**:
  - `util/math.h` — 数学函数和向量运算
  - `util/types.h` — 基础类型定义（`float3`, `float4`, `dual3` 等）
  - `util/system.h` — CPU 特性检测（`system_cpu_support_avx2()`，仅 CPU 端）
- **被引用**: Cycles 中几乎所有涉及几何变换的模块都依赖此文件，包括：
  - 场景层: `scene/object.h`, `scene/camera.h`, `scene/geometry.h`, `scene/shader_nodes.cpp`
  - 内核层: `kernel/svm/mapping_util.h`, `kernel/osl/services_gpu.h`
  - BVH 层: `bvh/unaligned.cpp`
  - 工具层: `util/texture.h`, `util/projection.h`, `util/boundbox.h`
  - 应用层: `app/cycles_xml.cpp`
  - 图节点: `graph/node.cpp`, `graph/node_type.cpp`

## 实现细节

1. **与 Embree 的精度一致性**: `transform_inverse_impl()` 的实现刻意与 Embree 光线追踪库的矩阵求逆逐位匹配（bit-exact），确保实例化对象的光线变换结果完全一致，避免因数值微差导致的光线追踪瑕疵
2. **多后端 SIMD 分派**: 变换函数通过条件编译支持四种代码路径：SSE2（Intel CPU）、Metal Shading Language（Apple GPU）、通用标量回退、以及通过运行时检测的 AVX2 路径
3. **AVX2 独立编译单元**: `transform_avx2.cpp` 存在的原因是 AVX2 代码需要特定的编译器标志（如 `-mavx2 -mfma`），将其隔离到独立文件中可避免影响其他代码的编译。运行时通过 `system_cpu_support_avx2()` 检测后才调用
4. **运动模糊插值策略**: 变换分解为旋转（四元数 Slerp）、平移（线性）和缩放（线性）三个独立通道进行插值，避免了直接对矩阵元素插值导致的非刚体变形问题
5. **宏别名**: 文件末尾定义了 `transform_point_auto`、`transform_direction_auto` 等宏别名，目前直接映射到对应函数，为未来可能的地址空间限定符适配预留接口

## 关联文件

- `src/util/transform.cpp` — CPU 端非内联函数实现（`transform_transposed_inverse`、`transform_motion_decompose` 等）
- `src/util/math.h` — 向量数学运算基础
- `src/util/types.h` — `float3`、`float4`、`dual3` 等类型定义
- `src/util/system.h` — CPU 特性检测
- `src/test/util_transform_test.cpp` — 变换函数单元测试
