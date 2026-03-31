# background.h - 环境光(背景光)采样

## 概述
本文件实现了 Cycles 渲染器中背景光(环境光/HDRI 光照)的采样与 PDF 计算。支持三种采样策略的多重重要性采样(MIS)混合：环境贴图采样、门户(portal)采样和太阳盘采样。通过 2D CDF（累积分布函数）对环境贴图进行重要性采样，同时利用场景中的门户几何体引导采样以提升室内场景的收敛效率。

## 核心函数

### background_map_sample()
- **签名**: `ccl_device float3 background_map_sample(KernelGlobals kg, const float2 rand, ccl_private float *pdf)`
- **功能**: 基于预计算的 2D CDF 对环境贴图进行重要性采样。先在边缘 CDF 中采样 V 方向，再在条件 CDF 中采样 U 方向，使用二分查找定位采样区间。返回采样方向和对应的 PDF。

### background_map_pdf()
- **签名**: `ccl_device float background_map_pdf(KernelGlobals kg, const float3 direction)`
- **功能**: 计算给定方向在环境贴图采样策略下的概率密度。将方向转换为等距矩形投影坐标后查询 CDF 表。

### background_portal_data_fetch_and_check_side()
- **签名**: `ccl_device_inline bool background_portal_data_fetch_and_check_side(KernelGlobals kg, const float3 P, const int index, ccl_private float3 *lightpos, ccl_private float3 *dir)`
- **功能**: 获取指定门户的位置和方向数据，并检查着色点是否位于门户的正确一侧（即门户朝向着色点的方向）。

### background_portal_pdf()
- **签名**: `ccl_device_inline float background_portal_pdf(KernelGlobals kg, const float3 P, float3 direction, const int ignore_portal, ccl_private bool *is_possible)`
- **功能**: 计算给定方向通过所有可见门户的混合 PDF。遍历所有门户，对圆形门户使用面积到立体角的转换，对矩形门户使用球面矩形采样 PDF。

### background_num_possible_portals()
- **签名**: `ccl_device int background_num_possible_portals(KernelGlobals kg, const float3 P)`
- **功能**: 统计从着色点 P 可见的门户数量（即位于门户正确一侧的门户数）。

### background_portal_sample()
- **签名**: `ccl_device float3 background_portal_sample(KernelGlobals kg, const float3 P, float2 rand, const int num_possible, ccl_private int *sampled_portal, ccl_private float *pdf)`
- **功能**: 从可见门户中随机选择一个并在其上采样方向。支持矩形门户和圆形门户的不同采样方式。

### background_sun_sample()
- **签名**: `ccl_device_inline float3 background_sun_sample(KernelGlobals kg, const float2 rand, ccl_private float *pdf)`
- **功能**: 在太阳盘的锥体内均匀采样一个方向。使用 `sample_uniform_cone` 实现。

### background_sun_pdf()
- **签名**: `ccl_device_inline float background_sun_pdf(KernelGlobals kg, const float3 D)`
- **功能**: 计算给定方向落在太阳盘锥体内的 PDF。

### background_light_sample()
- **签名**: `ccl_device_inline float3 background_light_sample(KernelGlobals kg, const float3 P, float2 rand, ccl_private float *pdf)`
- **功能**: 背景光的统一采样入口。根据预设权重在门户、太阳盘、环境贴图三种策略中选择一种进行采样，然后通过 MIS 混合所有策略的 PDF。当无可用策略时回退到均匀球面采样。

### background_light_pdf()
- **签名**: `ccl_device float background_light_pdf(KernelGlobals kg, const float3 P, float3 direction)`
- **功能**: 计算给定方向在背景光所有采样策略混合后的 PDF。是 `background_light_sample` 的对偶函数，用于前向路径追踪中的 MIS 权重计算。

### background_light_tree_parameters()
- **签名**: `template<bool in_volume_segment> ccl_device_forceinline bool background_light_tree_parameters(const float3 centroid, const float t, ccl_private float &cos_theta_u, ccl_private float2 &distance, ccl_private float3 &point_to_centroid, ccl_private float &theta_d)`
- **功能**: 为光源树的重要性计算提供背景光参数。背景光覆盖整个球面（cos_theta_u = -1），距离设为 1。

## 依赖关系
- **内部头文件**: `kernel/globals.h`, `kernel/camera/projection.h`, `kernel/light/area.h`, `kernel/light/common.h`, `util/math_intersect.h`
- **被引用**: `kernel/light/light.h`, `kernel/light/tree.h`

## 实现细节 / 关键算法
1. **2D CDF 环境贴图采样**: 使用预计算的边缘和条件 CDF 表（类似 PBRT 的方法），通过二分查找(`std::lower_bound` 风格)实现 O(log n) 的反转采样。CDF 表存储为 (函数值, 累积值) 的浮点对。
2. **三策略 MIS 混合**: 将随机数 `rand.x` 划分为三段区间，分别对应门户、太阳盘和环境贴图采样。选中某策略后，将该策略的 PDF 与其他策略的 PDF 按权重混合，实现无偏的多策略采样。
3. **门户采样**: 门户本质上是场景中标记为门户的面积光源，用于引导环境光采样穿过窗户等开口。对矩形门户复用 `area_light_rect_sample` 函数，对圆形门户使用椭圆采样加面积到立体角转换。
4. **等距矩形投影**: 使用 `equirectangular_to_direction` 和 `direction_to_equirectangular` 在方向和 UV 坐标之间转换，包含 sin(theta) 的雅可比行列式修正。

## 关联文件
- `kernel/light/area.h` - 门户采样复用面积光源的球面矩形采样
- `kernel/light/common.h` - 提供 `LightSample` 结构体和工具函数
- `kernel/light/light.h` - 在 `light_sample` 中调用 `background_light_sample`
- `kernel/light/tree.h` - 调用 `background_light_tree_parameters`
- `kernel/camera/projection.h` - 提供等距矩形投影函数
