# ao.h - 环境光遮蔽(AO)节点的SVM实现

## 概述

`ao.h` 实现了着色器虚拟机(SVM)中的环境光遮蔽(Ambient Occlusion)节点。该节点通过从着色点发射采样光线来估算几何遮蔽程度，可用于增强表面细节的阴影效果或作为着色器的输入因子。此功能依赖光线追踪能力，仅在启用 `__SHADER_RAYTRACE__` 编译选项时可用。

## 核心函数

### `svm_ao` / `__direct_callable__svm_node_ao`（OptiX 版本）
- **签名**: `ccl_device float svm_ao(KernelGlobals kg, ConstIntegratorState state, ShaderData *sd, float3 N, float max_dist, const int num_samples, const int flags)`
- **功能**: 核心 AO 采样计算函数，返回未遮蔽比例（0.0 = 完全遮蔽，1.0 = 完全无遮蔽）。
- **算法**:
  1. 提前退出条件检查：最大距离 <= 0、采样数 < 1、无对象、BVH 未构建
  2. 根据 `NODE_AO_INSIDE` 标志翻转法线方向
  3. 构建法线坐标系（正交基 T, B, N）
  4. 对每个采样：
     - 使用分支随机数生成均匀圆盘采样 `(d.x, d.y)`
     - 将圆盘采样映射到半球方向 `D = (d.x, d.y, sqrt(1-d^2))`
     - 转换到世界坐标系
     - 发射光线（P 到 P + max_dist * D）
     - 根据 `NODE_AO_ONLY_LOCAL` 标志选择：
       - 本地相交（仅检测同一对象）: `scene_intersect_local`
       - 全局阴影相交: `scene_intersect_shadow`
  5. 返回 `未遮蔽计数 / 总采样数`

### `svm_node_ao`
- **签名**: `template<uint node_feature_mask, typename ConstIntegratorGenericState> void svm_node_ao(...)`
- **功能**: SVM 节点入口函数，解包节点参数并调用核心 AO 计算。
- **输出**:
  - `out_ao_offset` — AO 因子（浮点值）
  - `out_color_offset` — AO * 颜色（float3）

## 依赖关系

- **内部头文件**:
  - `kernel/globals.h` — 全局内核数据
  - `kernel/integrator/path_state.h` — 路径状态与随机数
  - `kernel/bvh/bvh.h` — BVH 光线相交测试
  - `kernel/sample/mapping.h` — 采样映射函数（均匀圆盘采样）
  - `kernel/svm/util.h` — SVM 栈操作工具
- **被引用**: `kernel/osl/services.cpp`（OSL 着色器的 AO 实现也复用了类似逻辑）

## 实现细节 / 关键算法

1. **余弦加权半球采样**: 使用 `sample_uniform_disk` 生成圆盘上的均匀采样点 `(d.x, d.y)`，然后通过 `z = sqrt(1 - d^2)` 映射到半球。这种方法等效于余弦加权的半球采样，使得接近法线方向的光线被更多采样。

2. **全局半径支持**: 当设置 `NODE_AO_GLOBAL_RADIUS` 标志时，使用全局积分器设置的 AO 距离 (`kernel_data.integrator.ao_bounces_distance`) 替代节点参数中的距离。

3. **OptiX 直接可调用(Direct Callable)**: 在 OptiX 后端，AO 计算通过 `optixDirectCall<float>(0, ...)` 调用，索引 0 对应 AO 的 callable 程序，以满足 OptiX 的光线追踪管线要求。

4. **特征掩码守卫**: 使用 `IF_KERNEL_NODES_FEATURE(RAYTRACE)` 宏确保仅在内核启用了光线追踪节点特征时才执行 AO 计算，否则返回默认值 1.0。

## 关联文件

- `kernel/svm/svm.h` — SVM 主调度器（虽然 AO 的 include 在 `osl/services.cpp` 中）
- `kernel/bvh/bvh.h` — BVH 加速结构
- `kernel/svm/bevel.h` — 同样使用光线追踪的倒角节点
