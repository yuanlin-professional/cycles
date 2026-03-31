# ies.h - IES光域节点的SVM实现

## 概述

`ies.h` 实现了着色器虚拟机(SVM)中的 IES 光域(IES Light Profile)纹理节点。IES（Illuminating Engineering Society）光域文件描述了灯具在各方向上的光强分布。该节点将输入方向向量转换为球面坐标，然后在预加载的 IES 数据中进行插值查找，输出该方向上的光强因子。

## 核心函数

### `svm_node_ies`
- **签名**: `ccl_device_noinline void svm_node_ies(KernelGlobals kg, ShaderData *sd, float *stack, const uint4 node)`
- **功能**: IES 光域纹理节点的完整实现。
- **参数**:
  - `strength_offset` / `node.w` — 强度值（从栈或默认值获取）
  - `vector_offset` — 输入方向向量
  - `slot` (`node.z`) — IES 数据在内核中的槽位索引
  - `fac_offset` — 输出因子的栈偏移
- **算法**:
  1. 加载并归一化方向向量
  2. 计算垂直角度: `v_angle = acos(-vector.z)`（从 -Z 轴测量的天顶角）
  3. 计算水平角度: `h_angle = atan2(vector.x, vector.y) + pi`（映射到 [0, 2pi] 范围）
  4. 调用 `kernel_ies_interp(kg, slot, h_angle, v_angle)` 在 IES 数据中双线性插值
  5. 输出 `strength * ies_value`

## 依赖关系

- **内部头文件**:
  - `kernel/svm/util.h` — SVM 栈操作工具
  - `kernel/util/ies.h` — IES 数据插值函数 (`kernel_ies_interp`)
- **被引用**: `kernel/svm/svm.h`

## 实现细节 / 关键算法

1. **球面坐标映射**: 方向向量通过标准球面坐标转换。垂直角使用 `acos(-z)` 使得 Z 轴向下对应 0 度（光源正下方），Z 轴向上对应 180 度。水平角使用 `atan2(x, y) + pi` 将 Y 轴正方向映射为 0 度水平角。

2. **IES 数据插值**: `kernel_ies_interp` 函数在预处理后的 IES 数据表中执行双线性插值。IES 数据在场景加载阶段被解析为均匀角度网格，存储在内核数据中。

3. **默认值处理**: 强度参数使用 `stack_load_float_default` 加载，支持从栈读取动态值或使用节点中编码的静态默认值。

## 关联文件

- `kernel/util/ies.h` — IES 数据的内核插值实现
- `kernel/svm/svm.h` — SVM 主调度器
- Blender 端的 IES 文件解析器（场景加载阶段）
