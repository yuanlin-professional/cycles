# light.h - 光源采样与求交调度中心

## 概述
本文件是 Cycles 渲染器光源子系统的核心调度模块，汇聚了所有光源类型（面积光、背景光、远距光、点光源、聚光灯、三角形光）的采样和求交逻辑。提供统一的 `light_sample` 接口按光源类型分发采样调用，以及 `lights_intersect` 接口实现光线与所有灯光的批量求交测试。同时包含光源链接(light linking)和阴影链接(shadow linking)功能的实现。

## 核心函数

### light_select_reached_max_bounces()
- **签名**: `ccl_device_inline bool light_select_reached_max_bounces(KernelGlobals kg, const int index, const int bounce)`
- **功能**: 检查当前弹射次数是否超过指定光源的最大弹射次数限制。

### light_link_receiver_nee()
- **签名**: `ccl_device_inline int light_link_receiver_nee(KernelGlobals kg, const ccl_private ShaderData *sd)`
- **功能**: 获取下一事件估计(NEE)的光源链接接收者对象 ID。在启用光源链接特性时返回当前着色表面的对象 ID。

### light_link_receiver_forward()
- **签名**: `ccl_device_inline int light_link_receiver_forward(KernelGlobals kg, IntegratorState state)`
- **功能**: 获取前向路径追踪的光源链接接收者对象 ID。从积分器状态的 `mis_ray_object` 中获取。

### light_link_light_match()
- **签名**: `ccl_device_inline bool light_link_light_match(KernelGlobals kg, const int object_receiver, const int object_emitter)`
- **功能**: 检查光源发射体是否与接收者匹配（光源链接）。通过比较发射体的 `light_set_membership` 位掩码与接收者的 `receiver_light_set` 索引判断。

### light_link_object_match()
- **签名**: `ccl_device_inline bool light_link_object_match(KernelGlobals kg, const int object_receiver, const int object_emitter)`
- **功能**: 检查对象发射体是否与接收者匹配（光源链接）。世界体积(OBJECT_NONE)默认对所有对象可见。

### light_sample() (单灯版本)
- **签名**: `template<bool in_volume_segment> ccl_device_inline bool light_sample(KernelGlobals kg, const int lamp, const float2 rand, const float3 P, const float3 N, const int shader_flags, const uint32_t path_flag, ccl_private LightSample *ls)`
- **功能**: 在指定灯光上采样一个点。根据光源类型（远距光、背景光、聚光灯、点光源、面积光）分发到对应的采样函数。体积段中的远距光和背景光使用虚拟采样（实际采样在散射点处进行）。

### light_sample() (完整版本)
- **签名**: `template<bool in_volume_segment> ccl_device_noinline bool light_sample(KernelGlobals kg, const float3 rand_light, const float time, const float3 P, const float3 N, const int object_receiver, const int shader_flags, const int bounce, const uint32_t path_flag, ccl_private LightSample *ls)`
- **功能**: 完整的光源采样流程。从光源树或分布采样中获取发射体信息，执行光源链接检查，然后对网格光(三角形)或灯光调用具体的采样函数。最终 PDF 乘以选择 PDF。

### lights_intersect_impl()
- **签名**: `template<bool is_main_path> ccl_device_forceinline int lights_intersect_impl(KernelGlobals kg, const ccl_private Ray *ccl_restrict ray, ccl_private Intersection *ccl_restrict isect, ...)`
- **功能**: 光线与所有灯光的求交实现。遍历场景中的所有灯光，按类型调用 `spot_light_intersect`、`point_light_intersect`、`area_light_intersect`、`distant_light_intersect`。包含着色器可见性过滤、MNEE 裁剪、阴影捕捉器排除、阴影链接和光源链接等逻辑。

### lights_intersect()
- **签名**: `ccl_device bool lights_intersect(KernelGlobals kg, IntegratorState state, const ccl_private Ray *ccl_restrict ray, ccl_private Intersection *ccl_restrict isect, ...)`
- **功能**: 主路径的灯光求交接口。调用 `lights_intersect_impl<true>`，返回最近的灯光交点。

### lights_intersect_shadow_linked()
- **签名**: `ccl_device int lights_intersect_shadow_linked(KernelGlobals kg, const ccl_private Ray *ccl_restrict ray, ccl_private Intersection *ccl_restrict isect, ...)`
- **功能**: 阴影链接专用的灯光求交接口。调用 `lights_intersect_impl<false>`，使用蓄水池采样处理多个交点。

### light_sample_from_intersection()
- **签名**: `ccl_device bool light_sample_from_intersection(KernelGlobals kg, const ccl_private Intersection *ccl_restrict isect, const float3 ray_P, const float3 ray_D, const float3 N, const uint32_t path_flag, ccl_private LightSample *ccl_restrict ls)`
- **功能**: 从光线求交结果构建光源采样数据。根据光源类型（聚光灯、点光源、面积光）分发到对应的 `*_sample_from_intersection` 函数。用于前向路径追踪中的 MIS 计算。

## 依赖关系
- **内部头文件**: `kernel/geom/object.h`, `kernel/globals.h`, `kernel/integrator/state.h`, `kernel/light/area.h`, `kernel/light/background.h`, `kernel/light/distant.h`, `kernel/light/point.h`, `kernel/light/spot.h`, `kernel/light/triangle.h`, `kernel/sample/lcg.h`
- **被引用**: `kernel/light/sample.h`, `kernel/integrator/shade_volume.h`, `kernel/integrator/intersect_closest.h`, `kernel/integrator/intersect_dedicated_light.h`, `kernel/integrator/shade_background.h`, `kernel/integrator/shade_dedicated_light.h`, `kernel/integrator/shade_light.h`

## 实现细节 / 关键算法
1. **光源类型分发**: 使用 `LightType` 枚举和 if-else 链进行类型分发。远距光和背景光在体积段中使用虚拟采样（PDF=1, eval_fac=0），延迟到实际散射点处再求值。
2. **光源链接**: 通过 64 位集合成员位掩码实现。每个发射体有 `light_set_membership`，每个接收者有 `receiver_light_set` 索引（0-63），通过位运算 `(1 << receiver_set) & membership` 判断匹配。
3. **阴影链接**: 主路径与专用光照路径使用不同的求交逻辑。主路径排除具有非默认 `shadow_set_membership` 的间接光线，专用路径只处理这些被排除的灯光，使用 LCG 蓄水池采样随机选择交点。
4. **自相交避免**: 通过比较 `last_prim`、`last_object` 和 `last_type` 跳过与上一次交点重复的灯光。

## 关联文件
- `kernel/light/area.h` - 面积光源实现
- `kernel/light/background.h` - 背景光实现
- `kernel/light/distant.h` - 远距光实现
- `kernel/light/point.h` - 点光源实现
- `kernel/light/spot.h` - 聚光灯实现
- `kernel/light/triangle.h` - 三角形光源实现
- `kernel/light/sample.h` - 上层调用方
- `kernel/integrator/` - 积分器各阶段调用本文件
