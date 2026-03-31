# distribution.h - 光源分布采样(扁平CDF)

## 概述
本文件实现了 Cycles 渲染器中基于 CDF（累积分布函数）的简单光源分布采样。该方法不考虑着色点的位置或法线，按面积比例从场景中所有光源和发光三角形中选择一个进行采样。这是光源树(light tree)之外的备选采样策略，适用于禁用光源树或简单场景。

## 核心函数

### light_distribution_sample() (索引版本)
- **签名**: `ccl_device int light_distribution_sample(KernelGlobals kg, const float rand)`
- **功能**: 使用二分查找在预计算的光源分布 CDF 中定位给定随机数对应的光源索引。类似 PBRT 的 `std::upper_bound` 实现，按面积比例选择点光源或发光三角形。返回值经过 clamp 处理以防止浮点舍入误差。

### light_distribution_sample() (LightSample 版本)
- **签名**: `ccl_device_noinline bool light_distribution_sample(KernelGlobals kg, const float rand, ccl_private LightSample *ls)`
- **功能**: 光源分布采样的高层接口。调用索引版本获取光源索引后，设置 `ls->emitter_id` 和 `ls->pdf_selection`。选择 PDF 为所有光源的均匀概率 `distribution_pdf_lights`。

### light_distribution_pdf_lamp()
- **签名**: `ccl_device_inline float light_distribution_pdf_lamp(KernelGlobals kg)`
- **功能**: 返回基于分布采样的灯光选择 PDF。值为预计算的 `distribution_pdf_lights`，即 1/(灯光数量) 的等效值（考虑面积权重后的归一化因子）。

## 依赖关系
- **内部头文件**: `kernel/globals.h`, `kernel/light/common.h`
- **被引用**: `kernel/light/sample.h`

## 实现细节 / 关键算法
1. **CDF 二分查找**: 使用类似 `std::upper_bound` 的实现在 `light_distribution` 数组中查找。每次将搜索范围减半，比较随机数与 `totarea`（累积面积）来决定搜索方向。复杂度为 O(log N)，其中 N 为光源数量。
2. **面积比例采样**: CDF 按光源面积构建，使得大面积光源被选中的概率更高。但该方法未考虑光源功率，作者注释中指出按功率采样是潜在改进方向，但对于任意着色器难以准确定义。
3. **与光源树的互斥关系**: 此分布采样是光源树的简化替代方案。在 `sample.h` 中通过 `kernel_data.integrator.use_light_tree` 决定使用哪种策略。光源树考虑了着色点的空间位置和法线方向，理论上更优但计算开销更大。

## 关联文件
- `kernel/light/common.h` - 提供 `LightSample` 结构体
- `kernel/light/sample.h` - 唯一引用方，在 `light_sample_from_position` 和 `light_sample_from_volume_segment` 中调用
- `kernel/light/tree.h` - 互斥的替代光源选择方案
