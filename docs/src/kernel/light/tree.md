# tree.h - 光源树采样与重要性评估

## 概述
本文件实现了 Cycles 渲染器中基于光源树(light tree)的光源选择算法。该算法基于 Alejandro Conty Estevez 和 Christopher Kulla 的论文"Importance Sampling of Many Lights with Adaptive Tree Splitting"的改进版本。与原始论文不同，Cycles 不支持节点分裂（每个着色点只选择一个光源），因此使用最大和最小重要性的平均值作为选择概率，以减少方差。同时添加了对远距光的支持。

## 核心函数

### light_tree_cos_bound_subtended_angle()
- **签名**: `ccl_device float light_tree_cos_bound_subtended_angle(const KernelBoundingBox bbox, const float3 centroid, const float3 P)`
- **功能**: 计算节点包围盒的最小包围球对着色点的张角余弦 cos(theta_u)。当着色点在包围球内部时返回 -1（覆盖整个球面）。

### compute_v()
- **签名**: `ccl_device float3 compute_v(const float3 centroid, const float3 P, const float3 D, const float3 bcone_axis, const float t)`
- **功能**: 计算论文图 8 中的向量 v，即光线段上与发射体质心形成最小角度的方向。用于体积渲染中的重要性评估。

### is_light() / is_mesh() / is_triangle() / is_leaf()
- **功能**: 发射体类型判断工具函数。通过 `kemitter->light.id` 的符号和 `kemitter->object_id` 区分灯光、网格和三角形。叶节点包括常规叶节点和远距光节点。

### light_tree_to_local_space()
- **签名**: `template<bool in_volume_segment> ccl_device void light_tree_to_local_space(KernelGlobals kg, const int object_id, ccl_private float3 &P, ccl_private float3 &N_or_D, ccl_private float &t)`
- **功能**: 将光线/法线从世界空间变换到网格光源的局部空间。支持运动模糊变换，体积段模式下变换方向并更新距离。

### light_tree_importance()
- **签名**: `template<bool in_volume_segment> ccl_device void light_tree_importance(const float3 N_or_D, const bool has_transmission, const float3 point_to_centroid, const float cos_theta_u, const KernelBoundingCone bcone, const float max_distance, const float min_distance, const float energy, const float theta_d, ccl_private float &max_importance, ccl_private float &min_importance)`
- **功能**: 光源树重要性计算的核心函数。同时计算上界(max)和下界(min)重要性。上界使用保守估计（最小入射角、最小出射角、最小距离），下界使用宽松估计（最大入射角、最大出射角、最大距离）。不透明表面背后的节点重要性为零。

### light_tree_node_importance()
- **签名**: `template<bool in_volume_segment> ccl_device void light_tree_node_importance(const float3 P, const float3 N_or_D, const float t, const bool has_transmission, const ccl_global KernelLightTreeNode *knode, ccl_private float &max_importance, ccl_private float &min_importance)`
- **功能**: 计算光源树内部节点的重要性。从节点的包围盒和包围锥提取几何参数，处理远距光节点和常规节点的不同逻辑。体积段模式下使用射线到质心的最近点计算距离和角度。

### light_tree_emitter_importance()
- **签名**: `template<bool in_volume_segment> ccl_device void light_tree_emitter_importance(KernelGlobals kg, const float3 P, const float3 N_or_D, const float t, const bool has_transmission, const int emitter_index, ccl_private float &max_importance, ccl_private float &min_importance)`
- **功能**: 计算单个发射体的重要性。根据发射体类型（网格、灯光、三角形）获取质心和方向，然后调用各光源类型的 `*_tree_parameters` 函数获取几何参数，最终调用 `light_tree_importance`。

### light_tree_child_importance()
- **签名**: `template<bool in_volume_segment> ccl_device void light_tree_child_importance(KernelGlobals kg, const float3 P, const float3 N_or_D, const float t, const bool has_transmission, const ccl_global KernelLightTreeNode *knode, ccl_private float &max_importance, ccl_private float &min_importance)`
- **功能**: 计算子节点的重要性。当节点只有一个发射体时退化为发射体重要性计算。

### sample_reservoir()
- **签名**: `ccl_device void sample_reservoir(const int current_index, const float current_weight, ccl_private int &selected_index, ccl_private float &selected_weight, ccl_private float &total_weight, ccl_private float &rand)`
- **功能**: 蓄水池采样(reservoir sampling)的核心实现。以与权重成比例的概率选择元素，支持流式处理。包含 `-ffast-math` 的鲁棒性处理。

### light_tree_cluster_select_emitter()
- **签名**: `template<bool in_volume_segment> ccl_device int light_tree_cluster_select_emitter(KernelGlobals kg, ccl_private float &rand, ccl_private float3 &P, ccl_private float3 &N_or_D, ccl_private float &t, const bool has_transmission, ccl_private int *node_index, ccl_private float *pdf_factor)`
- **功能**: 从叶节点中使用蓄水池采样选择一个发射体。维护两个蓄水池（上界和下界），随机选择一个进行采样，PDF 为两者的平均。对网格发射体会切换到局部空间并返回子树节点。

### get_left_probability()
- **签名**: `template<bool in_volume_segment> ccl_device bool get_left_probability(KernelGlobals kg, const float3 P, const float3 N_or_D, const float t, const bool has_transmission, const int left_index, const int right_index, ccl_private float &left_probability)`
- **功能**: 计算选择左子节点的概率。分别计算左右子节点的上下界重要性，概率为上界概率和下界概率的平均值。两个子节点重要性均为零时返回 false。

### light_tree_root_node_index()
- **签名**: `ccl_device int light_tree_root_node_index(KernelGlobals kg, const int object_receiver)`
- **功能**: 获取光源树的根节点索引。支持光源链接特性，不同接收者集可有不同的根节点。

### light_tree_sample()
- **签名**: `template<bool in_volume_segment> ccl_device_noinline bool light_tree_sample(KernelGlobals kg, const float rand, const float3 P, float3 N_or_D, float t, const int object_receiver, const int shader_flags, ccl_private LightSample *ls)`
- **功能**: 光源树采样的主入口。从根节点开始自顶向下遍历，在每个内部节点使用蓄水池采样选择左或右子树，在叶节点使用蓄水池采样选择发射体。支持网格光源的二次遍历（先在顶层树选择网格，再在子树中选择三角形）。

### light_tree_pdf()
- **签名**: `template<bool in_volume_segment> ccl_device float light_tree_pdf(KernelGlobals kg, float3 P, float3 N, const float dt, const int path_flag, const int object_emitter, const uint index_emitter, const int object_receiver)`
- **功能**: 计算光源树选择给定发射体的 PDF。通过比特轨迹(bit_trail)从根到目标叶节点遍历，累乘每一级的选择概率。支持三角形发射体的两级遍历（顶层树 + 网格子树）。

### light_tree_pdf() (非模板版本)
- **签名**: `ccl_device float light_tree_pdf(KernelGlobals kg, float3 P, const float3 N, const float dt, const int path_flag, const int emitter_object, const uint emitter_id, const int object_receiver)`
- **功能**: PDF 计算的统一入口。根据路径标志判断是表面还是体积散射，分发到对应的模板版本。

## 依赖关系
- **内部头文件**: `kernel/light/area.h`, `kernel/light/background.h`, `kernel/light/common.h`, `kernel/light/distant.h`, `kernel/light/point.h`, `kernel/light/spot.h`, `kernel/light/triangle.h`, `util/math_fast.h`
- **被引用**: `kernel/light/sample.h`（条件编译 `__LIGHT_TREE__`）

## 实现细节 / 关键算法
1. **双蓄水池采样**: 核心创新在于同时维护上界和下界两个重要性蓄水池。采样时随机选择一个蓄水池进行采样（概率各 50%），最终 PDF 为两个蓄水池选中概率的平均值。这比仅使用保守上界更稳定，避免了方差过大的问题。
2. **重要性函数**: 重要性 = `|f_a * cos(theta_i') * energy * cos(theta') / d^2|`。其中 `theta_i'` 是入射角的下界/上界，`theta'` 是出射角的下界/上界，`d` 是距离的上界/下界。不可见节点（背面、超出锥角范围）重要性为零。
3. **比特轨迹(bit trail)**: 每个发射体预存储一个位编码路径，指示从根到叶节点的左右选择序列。PDF 计算时沿此路径遍历，每级乘以对应的选择概率。`bit_skip` 处理压缩跳过的层级。
4. **网格光源二级遍历**: 对于发光网格，光源树有两级结构——顶层树包含网格作为叶节点，每个网格有自己的子树包含其三角形。采样和 PDF 计算都需要处理这种二级结构，包括到局部空间的坐标变换。
5. **体积段估计**: 体积渲染使用公式 (4) 的 `(theta_b - theta_a) / d` 替代 `1/d^2`，其中 `theta_b - theta_a` 通过 `atan2` 计算射线段对质心的张角。

## 关联文件
- `kernel/light/area.h` - 提供 `area_light_tree_parameters`
- `kernel/light/background.h` - 提供 `background_light_tree_parameters`
- `kernel/light/distant.h` - 提供 `distant_light_tree_parameters`
- `kernel/light/point.h` - 提供 `point_light_tree_parameters`
- `kernel/light/spot.h` - 提供 `spot_light_tree_parameters`
- `kernel/light/triangle.h` - 提供 `triangle_light_tree_parameters`
- `kernel/light/sample.h` - 唯一引用方，在光源选择和 MIS PDF 计算中调用
