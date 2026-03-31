# svm.h - 着色器虚拟机(SVM)主解释器入口

## 概述
`svm.h` 是 Cycles 渲染器着色器虚拟机(SVM)的主入口文件。它汇总包含了所有 SVM 节点的头文件，并实现了核心的解释器主循环 `svm_eval_nodes`。着色器被编码为一系列 `uint4` 指令存储在一维纹理中，解释器逐条读取并执行这些指令，通过栈(stack)在节点之间传递浮点数据（因子、颜色、向量）。

着色器执行的最终结果是一个闭包(Closure)，包括闭包类型、关联标签、数据和权重。多闭包采样通过混合闭包节点实现，相关逻辑主要由 SVM 编译器处理。

## 核心函数

### `svm_eval_nodes<node_feature_mask, type, ConstIntegratorGenericState>`
- **签名**: `ccl_device void svm_eval_nodes(KernelGlobals kg, ConstIntegratorGenericState state, ccl_private ShaderData *sd, ccl_global float *render_buffer, const uint32_t path_flag)`
- **功能**: SVM 主解释器循环。分配固定大小的浮点栈 `stack[SVM_STACK_SIZE]`，从 `ShaderData` 中获取着色器偏移量，进入 `while(true)` 循环逐条读取并分发执行节点指令。
- **模板参数**:
  - `node_feature_mask`: 内核特性掩码，用于编译期裁剪不需要的节点特性
  - `type`: 着色器类型（`ShaderType`），控制 `NODE_SHADER_JUMP` 的跳转目标
  - `ConstIntegratorGenericState`: 积分器状态类型

### `SVM_CASE` 宏
- 当定义了 `__KERNEL_USE_DATA_CONSTANTS__` 时，会在 case 分支前检查 `kernel_data_svm_usage_##node` 标志，若节点未被使用则跳过执行，实现运行时节点裁剪优化。

## 依赖关系
- **内部头文件**:
  - `kernel/globals.h` - 内核全局数据
  - `kernel/types.h` - 内核类型定义
  - `kernel/svm/types.h` - SVM 类型定义
  - `kernel/svm/util.h` - SVM 工具函数
  - 以及所有具体节点实现头文件（约 40+ 个，如 `aov.h`, `attribute.h`, `brick.h`, `checker.h`, `closure.h`, `noise.h`, `voronoi.h` 等）
- **被引用**: 被内核着色器求值入口调用（如 `kernel/integrator/shade_surface.h` 等积分器模块）

## 实现细节

### 指令编码格式
- 每条指令编码为一个或多个 `uint4`（即 4 个 `uint` 分量 x/y/z/w）
- `node.x` 存储节点类型枚举值，`node.y/z/w` 存储参数或栈偏移量
- 浮点数通过 `__uint_as_float` / `__int_as_float` 进行类型双关编码

### 着色器跳转机制
- `NODE_SHADER_JUMP` 根据着色器类型（表面/体积/位移）跳转到对应的指令段
- `NODE_JUMP_IF_ZERO` / `NODE_JUMP_IF_ONE` 实现条件跳转，用于混合闭包优化

### 条件编译节点
- `__HAIR__`: 启用毛发信息节点
- `__POINTCLOUD__`: 启用点云信息节点
- `__SHADER_RAYTRACE__`: 启用环境光遮蔽(AO)和倒角(Bevel)节点

### GPU 栈存储
栈数据存储在 GPU 的本地内存中（而非寄存器），因为索引方式在编译期无法确定。当相同着色器被并行执行时，内存访问可以被合并和缓存，一定程度上弥补了性能损失。

## 关联文件
- `kernel/svm/types.h` - 节点类型枚举和闭包类型定义
- `kernel/svm/util.h` - 栈操作和节点读取工具函数
- `kernel/svm/node_types_template.h` - 节点类型 X-Macro 模板
- 所有 `kernel/svm/*.h` 节点实现文件
