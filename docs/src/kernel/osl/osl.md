# osl.h - 开放着色语言(OSL)着色器引擎主入口

## 概述

`osl.h` 是 Cycles 渲染器中开放着色语言(OSL)着色器引擎的主要头文件,提供了将 Cycles 着色数据转换为 OSL 着色器全局变量的接口、闭包树扁平化函数,以及在 CPU 和 GPU (OptiX) 上执行各类着色器(表面、体积、位移)的模板函数 `osl_eval_nodes`。该文件是 OSL 集成层的核心枢纽,将 OSL 的着色器求值结果转化为 Cycles 内部的闭包/双向散射分布函数(BSDF)数据。

## 类与结构体

本文件无独立类或结构体定义,但核心函数操作以下类型:
- **`ShaderGlobals`**: OSL 着色器全局变量(定义于 `types.h`)
- **`ShaderData`**: Cycles 内部着色数据结构
- **`OSLClosure`**: OSL 闭包树节点基类(定义于 `types.h`)

## 核心函数

### `shaderdata_to_shaderglobals`
```cpp
ccl_device_inline void shaderdata_to_shaderglobals(
    ccl_private ShaderData *sd,
    const uint32_t path_flag,
    ccl_private ShaderGlobals *globals)
```
- **功能**: 将 Cycles 内部的 `ShaderData` 结构体全面映射到 OSL 的 `ShaderGlobals`。映射内容包括:
  - 位置 P 及其偏导数(从紧凑微分格式展开)
  - 入射方向 I 及其偏导数
  - 着色法线 N、几何法线 Ng
  - UV 坐标 (u, v) 及其偏导数
  - 表面切线偏导数 dPdu, dPdv
  - 时间、光线类型(raytype = path_flag)、面朝方向(backfacing)
  - ShaderData 指针存入 `sd` 字段供渲染服务回调使用
  - `shader2common` 和 `object2common` 设为 `sd` 指针(矩阵变换由服务层处理)
  - `Ci` 闭包输出初始化为 `nullptr`

### `flatten_closure_tree`
```cpp
ccl_device void flatten_closure_tree(
    KernelGlobals kg,
    ccl_private ShaderData *sd,
    const uint32_t path_flag,
    const ccl_private OSLClosure *closure)
```
- **功能**: 将 OSL 着色器输出的闭包树递归结构转化为 Cycles 可用的扁平 BSDF 列表。详见"实现细节"章节。

### `osl_eval_nodes` (模板函数)
```cpp
// CPU 版本
template<ShaderType type>
void osl_eval_nodes(const ThreadKernelGlobalsCPU *kg,
                    const void *state, ShaderData *sd, uint32_t path_flag);

// GPU 版本
template<ShaderType type, typename ConstIntegratorGenericState>
ccl_device_inline void osl_eval_nodes(KernelGlobals kg,
    ConstIntegratorGenericState state,
    ccl_private ShaderData *sd, const uint32_t path_flag);
```
- **功能**: 根据着色器类型(表面、体积、位移)执行对应的 OSL 着色器。
- **CPU 版本**: 仅声明模板原型,特化实现在 `closures.cpp` 中。
- **GPU 版本** (OptiX): 内联实现,通过 `optixDirectCall` 执行着色器。对于表面着色器,在主着色器执行前处理自动凹凸映射。着色器索引计算公式为: `2 + 1 + (shader + type * kernel_data.max_shaders)`。

## 依赖关系

- **内部头文件**:
  - `kernel/osl/closures_setup.h` - 闭包设置函数
  - `kernel/osl/types.h` - OSL 类型定义
  - `kernel/util/differential.h` - 微分工具函数
  - `kernel/geom/attribute.h`, `kernel/geom/primitive.h` - 几何属性(仅 OptiX)
- **被引用**:
  - `src/kernel/osl/closures.cpp` - 使用 `shaderdata_to_shaderglobals` 和 `flatten_closure_tree`
  - `src/kernel/osl/services_gpu.h` - GPU 渲染服务中使用本文件

## 实现细节 / 关键算法

### 闭包树扁平化

`flatten_closure_tree` 使用迭代方式(非递归)遍历闭包树,避免了栈溢出风险:

1. **栈结构**: 固定大小为 16 的两个并行栈(`weight_stack` 和 `closure_stack`),限制了闭包树的最大嵌套深度。
2. **遍历策略**:
   - **乘法节点** (`MUL`): 将权重累乘,直接前进到子闭包
   - **加法节点** (`ADD`): 右子树入栈(保存当前权重),遍历左子树
   - **分层节点** (`LAYER`): 底层入栈并标记栈层级,遍历顶层;顶层处理完毕后,用累积的反照率调整底层权重
   - **叶子节点**: 调用对应的 `osl_closure_*_setup` 函数
3. **层级处理**: `layer_stack_level` 标记当前分层闭包在栈中的位置。当从栈中弹出该层级时,通过 `closure_layering_weight` 函数计算底层的有效权重。

### GPU 端 OptiX 着色器调用

GPU 版本使用 OptiX 的可调用程序(Callable Programs)机制:
- 使用 1024 字节的闭包池(`closure_pool`)存储闭包数据
- `shade_index` 编码路径状态:`state + 1` 表示正常路径,`-state - 1` 表示阴影路径
- 可调用程序索引 = `NUM_CALLABLE_PROGRAM_GROUPS(2) + 1(相机程序) + shader + type * max_shaders`

### 位移着色器的特殊处理

位移着色器执行后,直接从 `globals.P` 回读修改后的位置到 `sd->P`。与表面/体积着色器不同,位移着色器不产生闭包输出。

## 关联文件

- `src/kernel/osl/closures.cpp` - 包含 CPU 端 `osl_eval_nodes` 的模板特化实现
- `src/kernel/osl/closures_setup.h` - 闭包设置函数(被 `flatten_closure_tree` 调用)
- `src/kernel/osl/closures_template.h` - 闭包类型定义(通过 X-Macro 生成 switch-case)
- `src/kernel/osl/types.h` - `ShaderGlobals`、`OSLClosure` 等核心类型
- `src/kernel/osl/camera.h` - 相机着色器单独处理
