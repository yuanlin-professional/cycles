# ies.h - IES 光域网数据插值

## 概述
本文件实现了 IES（Illuminating Engineering Society）光域网数据的内核端插值查询功能。IES 文件描述了光源在球面坐标系下各方向的光强分布，广泛用于建筑可视化中的灯具模拟。文件提供了基于三次插值的高质量光强查询函数，支持水平和垂直方向的环绕（wrap-around）处理，确保在极点区域和 360 度覆盖范围内的平滑过渡。

## 类与结构体
本文件未定义类或结构体。

## 枚举与常量
本文件未定义枚举或常量。

## 核心函数

### interpolate_ies_vertical()
- **签名**: `ccl_device_inline float interpolate_ies_vertical(KernelGlobals kg, const int ofs, const bool wrap_vlow, const bool wrap_vhigh, const int v, const int v_num, const float v_frac, const int h)`
- **功能**: 对 IES 数据在垂直方向（v 方向，对应天顶角）进行三次样条插值。查询指定水平角度 `h` 处相邻四个垂直采样点的光强值，并使用 `cubic_interp()` 进行插值。在北极（v_low）和南极（v_high）附近支持环绕处理以避免伪影——假设光源在极点附近具有对称性，使用对称位置的值替代越界查询。

### kernel_ies_interp()
- **签名**: `ccl_device_inline float kernel_ies_interp(KernelGlobals kg, const int slot, const float h_angle, const float v_angle)`
- **功能**: IES 光强查询的主入口函数。给定水平角度 `h_angle` 和垂直角度 `v_angle`（均为弧度），返回该方向的光强值。函数首先从全局数据表中定位 IES 数据偏移量，获取水平/垂直角度网格的维度，然后通过线性搜索找到对应的网格单元，最后在水平和垂直两个维度上分别进行三次插值。

## 依赖关系
- **内部头文件**:
  - `kernel/globals.h` — 提供 `KernelGlobals` 及 `kernel_data_fetch` 数据访问宏
- **被引用**:
  - `kernel/svm/ies.h` — 着色器虚拟机（SVM）中的 IES 纹理节点
  - `kernel/osl/services_gpu.h` — GPU 端 OSL 着色器服务
  - `kernel/osl/services.cpp` — CPU 端 OSL 着色器服务

## 实现细节 / 关键算法

### IES 数据存储格式
IES 数据以扁平化浮点数组的形式存储在 `kernel_data_fetch(ies, ...)` 全局查找表中。每个 IES 光域网的数据布局为：
1. `h_num`（水平角度数量，以 int 编码为 float）
2. `v_num`（垂直角度数量，以 int 编码为 float）
3. `h_num` 个水平角度值（弧度，升序排列）
4. `v_num` 个垂直角度值（弧度，升序排列）
5. `h_num * v_num` 个光强值（按水平优先存储）

### 双三次插值
插值在水平和垂直两个维度上分别使用三次样条（cubic interpolation），总共需要查询 4x4 = 16 个采样点。在每个维度上使用 `cubic_interp(a, b, c, d, frac)` 函数进行插值。

### 边界环绕处理
- **水平环绕**：当水平角度覆盖完整 360 度范围时（`h_low ~= 0` 且 `h_high ~= 2*pi`），在边界处使用对侧的值进行插值，实现无缝环绕。由于首尾条目值相同（360 度 = 0 度），环绕时需要跳到倒数第二个或第二个条目。
- **垂直环绕**：在北极（v ~= 0）和南极（v ~= pi）处分别判断是否环绕。环绕时利用光源对称性假设，使用极点对侧对称位置的值。
- 三次插值可能产生负值，最终结果通过 `max(..., 0.0f)` 钳制。

### 角度查找
当前使用线性搜索定位角度所在的网格单元。代码注释中提到可以考虑使用二分搜索优化，但对于绝大多数 IES 文件（角度数量有限）而言，线性搜索已足够高效。

## 关联文件
- `src/kernel/globals.h` — 全局数据访问
- `src/kernel/svm/ies.h` — SVM 中调用本文件进行 IES 查询的节点实现
- `src/scene/light.cpp` — 主机端 IES 数据的解析和上传逻辑
