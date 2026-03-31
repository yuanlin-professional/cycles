# displacement_shader.h - 位移着色器求值

## 概述

本文件提供了位移着色器的求值功能，用于在渲染过程中修改表面几何体的顶点位置（`sd->P`）。位移着色器通过执行 SVM（着色器虚拟机）或 OSL（Open Shading Language）节点来计算位移偏移量。该文件是 Cycles 渲染器积分器模块的一部分，支持 GPU 和 CPU 两种执行路径。

## 核心函数

### displacement_shader_eval()

- **签名**: `template<typename ConstIntegratorGenericState> ccl_device void displacement_shader_eval(KernelGlobals kg, ConstIntegratorGenericState state, ccl_private ShaderData *sd)`
- **功能**: 对给定的着色数据执行位移着色器求值。函数首先将闭包计数清零（`num_closure` 和 `num_closure_left` 置为 0），然后根据编译配置选择 OSL 或 SVM 后端执行位移节点。执行结果会直接修改 `sd->P`（表面位置），从而实现几何位移效果。
- **模板参数**: `ConstIntegratorGenericState` - 积分器泛型状态类型，支持不同上下文的调用。
- **条件编译**:
  - `__OSL__`: 若启用 OSL 且内核特性包含 `KERNEL_FEATURE_OSL_SHADING`，则使用 `osl_eval_nodes<SHADER_TYPE_DISPLACEMENT>` 求值。
  - `__SVM__`: 否则使用 `svm_eval_nodes<KERNEL_FEATURE_NODE_MASK_DISPLACEMENT, SHADER_TYPE_DISPLACEMENT>` 求值。

## 依赖关系

- **内部头文件**:
  - `kernel/globals.h` - 内核全局数据定义
  - `kernel/svm/svm.h` - SVM 着色器虚拟机（条件编译 `__SVM__`）
  - `kernel/osl/osl.h` - OSL 着色语言支持（条件编译 `__OSL__`）
- **被引用**:
  - `kernel/bake/bake.h` - 烘焙模块中使用位移着色器求值

## 实现细节

1. **闭包清零**: 位移着色器不产生 BSDF 闭包，因此在求值前将 `num_closure` 和 `num_closure_left` 均设为 0。
2. **双后端选择**: 运行时根据 `kernel_data.kernel_features` 中的 `KERNEL_FEATURE_OSL_SHADING` 标志位决定使用 OSL 还是 SVM 后端。OSL 优先级更高。
3. **就地修改**: 位移计算结果直接写入 `sd->P`，调用方无需额外处理返回值。

## 关联文件

- `kernel/svm/svm.h` - SVM 节点求值引擎
- `kernel/osl/osl.h` - OSL 节点求值引擎
- `kernel/bake/bake.h` - 烘焙流程中调用位移着色器
