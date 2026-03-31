# light_path.h - 光线路径与光线衰减节点的SVM实现

## 概述

`light_path.h` 实现了着色器虚拟机(SVM)中的光线路径(Light Path)节点和光线衰减(Light Falloff)节点。光线路径节点允许着色器查询当前光线的类型和状态信息（如是否为相机光线、阴影光线、反射深度等），常用于创建针对不同光线类型的条件着色逻辑。光线衰减节点则控制灯光的距离衰减方式。

## 核心函数

### `svm_node_light_path`
- **签名**: `template<uint node_feature_mask, typename ConstIntegratorGenericState> void svm_node_light_path(KernelGlobals kg, ConstIntegratorGenericState state, const ShaderData *sd, float *stack, const uint type, const uint out_offset, const uint32_t path_flag)`
- **功能**: 查询当前光线路径的各种属性，输出 float 值（通常为 0.0 或 1.0 的布尔值，或深度/长度等数值）。
- **支持的查询类型** (`NodeLightPath` 枚举):
  - `NODE_LP_camera` — 是否为相机光线
  - `NODE_LP_shadow` — 是否为阴影光线
  - `NODE_LP_diffuse` — 是否为漫反射光线
  - `NODE_LP_glossy` — 是否为光泽光线
  - `NODE_LP_singular` — 是否为奇异光线（delta 分布）
  - `NODE_LP_reflection` — 是否为反射光线
  - `NODE_LP_transmission` — 是否为透射光线
  - `NODE_LP_volume_scatter` — 是否为体积散射光线
  - `NODE_LP_backfacing` — 是否为背面
  - `NODE_LP_ray_length` — 光线长度
  - `NODE_LP_ray_depth` — 光线弹射深度
  - `NODE_LP_ray_transparent` — 透明弹射深度
  - `NODE_LP_ray_diffuse` — 漫反射弹射深度
  - `NODE_LP_ray_glossy` — 光泽弹射深度
  - `NODE_LP_ray_transmission` — 透射弹射深度
  - `NODE_LP_ray_portal` — 传送门弹射次数

### `svm_node_light_falloff`
- **签名**: `ccl_device_noinline void svm_node_light_falloff(ShaderData *sd, float *stack, const uint4 node)`
- **功能**: 光线衰减节点，控制灯光的距离衰减方式。
- **衰减类型** (`NodeLightFalloff` 枚举):
  - `NODE_LIGHT_FALLOFF_QUADRATIC` — 二次衰减（物理默认，不做修改）
  - `NODE_LIGHT_FALLOFF_LINEAR` — 线性衰减: `strength *= ray_length`
  - `NODE_LIGHT_FALLOFF_CONSTANT` — 恒定（无衰减）: `strength *= ray_length^2`
- **平滑参数**: 可选的 `smooth` 参数，在近距离防止强度趋于无穷: `strength *= dist^2 / (smooth + dist^2)`

## 依赖关系

- **内部头文件**:
  - `kernel/svm/util.h` — SVM 栈操作工具
- **被引用**: `kernel/svm/svm.h`

## 实现细节 / 关键算法

1. **路径标志查询**: 大部分光线类型查询通过位运算 `path_flag & PATH_RAY_*` 实现，返回 0.0 或 1.0。`backfacing` 查询使用 `sd->flag & SD_BACKFACING`，`ray_length` 直接读取 `sd->ray_length`。

2. **弹射深度查询**: 深度查询使用 `integrator_state_*_bounce` 系列函数从积分器状态中读取。需要 `LIGHT_PATH` 节点特征才能访问积分器状态，否则返回 0。对于 `ray_depth`，在阴影和发光评估路径上额外加 1（因为从表面/体积评估时实际上多了一次弹射）。

3. **远距灯光处理**: `light_falloff` 节点在 `ray_length == FLT_MAX`（远距灯光）时直接返回原始强度，因为非二次衰减模式在无穷远距离时会产生溢出。

4. **衰减模式的补偿逻辑**: 线性和恒定衰减通过乘以 `ray_length` 或 `ray_length^2` 来补偿 Cycles 默认的二次衰减。实际效果：恒定 = `strength * r^2 / r^2 = strength`；线性 = `strength * r / r^2 = strength / r`。

## 关联文件

- `kernel/svm/svm.h` — SVM 主调度器
- `kernel/integrator/path_state.h` — 路径弹射计数函数
