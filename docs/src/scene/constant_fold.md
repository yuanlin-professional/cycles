# constant_fold.h / constant_fold.cpp - 着色器图常量折叠优化器

## 概述

本文件实现了 Cycles 渲染器着色器图的常量折叠优化器。`ConstantFolder` 类在着色器图编译前的优化阶段工作，用于检测输入为常量的节点并将其输出替换为常量值或直接绕过（bypass）到上游节点。该优化器由 `ShaderGraph::constant_fold()` 按拓扑序调用，对每个节点的每个输出执行折叠尝试，可显著减少运行时着色器虚拟机(SVM)指令数量和栈使用。

## 类与结构体

### ConstantFolder

- **继承**: 无
- **功能**: 着色器节点常量折叠工具类。针对每个节点的每个输出插口创建一个实例，提供各种折叠辅助方法。节点类在其 `constant_fold()` 虚方法中调用这些方法来实现自己的折叠逻辑。
- **关键成员**:
  - `graph` — 所属着色器图（ShaderGraph* const）
  - `node` — 当前处理的节点（ShaderNode* const）
  - `output` — 当前处理的输出插口（ShaderOutput* const）
  - `scene` — 场景指针
- **关键方法**:

#### 常量值替换
  - `make_constant(float/float3/int)` — 将输出替换为常量值：将常量值写入所有下游输入插口，标记 `constant_folded_in`，然后断开输出连接
  - `make_constant_clamp(value, clamp)` — 带可选钳制的常量替换
  - `make_zero()` — 替换为零值（自动检测 float 或 float3）
  - `make_one()` — 替换为一值

#### 节点绕过
  - `bypass(ShaderOutput*)` — 将输出的所有下游连接重新链接到另一个输出插口，跳过当前节点
  - `discard()` — 丢弃闭包节点（断开输出连接）
  - `bypass_or_discard(ShaderInput*)` — 对闭包输入：有链接则绕过，无链接则丢弃
  - `try_bypass_or_make_constant(ShaderInput*, clamp)` — 综合尝试：输入已链接且无钳制则绕过，输入未链接则用其值替换为常量。返回 true 表示成功折叠

#### 输入值测试
  - `all_inputs_constant()` — 检查节点所有输入是否均为常量（无连接）
  - `is_zero(ShaderInput*)` — 检查输入是否为零值
  - `is_one(ShaderInput*)` — 检查输入是否为一值

#### 特定节点类型折叠
  - `fold_mix(NodeMix, clamp)` — 混合颜色节点折叠（旧版 MixNode，使用 Fac/Color1/Color2 插口）
  - `fold_mix_color(NodeMix, clamp_factor, clamp)` — 混合颜色节点折叠（新版 MixColorNode，使用 Factor/A/B 插口）
  - `fold_mix_float(clamp_factor, clamp)` — 混合浮点节点折叠
  - `fold_math(NodeMathType)` — 数学节点折叠
  - `fold_vector_math(NodeVectorMathType)` — 向量数学节点折叠
  - `fold_mapping(NodeMappingType)` — 映射节点折叠

## 核心函数

### fold_mix / fold_mix_color
颜色混合节点的常量折叠规则：
- **因子为 0**: 结果等于 Color1/A（对部分模式如 Light/Dodge/Burn 除外）
- **Blend 模式**: Color1 == Color2 时绕过其一；因子为 1 时使用 Color2/B
- **Add 模式**: 0 + X (fac=1) = X；X + 0 = X
- **Sub 模式**: X - 0 = X；X - X (fac=1) = 0
- **Mul 模式**: X * 1 = X；1 * X (fac=1) = X；X * 0 = 0；0 * X = 0
- **Div 模式**: X / 1 = X；0 / X = 0

### fold_math
标量数学节点的常量折叠规则：
- **Add**: X + 0 = X；0 + X = X
- **Subtract**: X - 0 = X
- **Multiply**: X * 1 = X；1 * X = X；X * 0 = 0；0 * X = 0
- **Divide**: X / 1 = X；0 / X = 0
- **Power**: 1^X = 1；X^0 = 1；X^1 = X

### fold_vector_math
向量数学节点的常量折叠规则：
- **Add/Subtract**: 与标量类似
- **Multiply/Divide**: 与标量类似，但按分量运算
- **Dot/Cross Product**: 任一输入为零则结果为零
- **Length/Absolute**: 零向量的长度/绝对值为零
- **Scale**: X * 0 = 0；X * 1 = X

### fold_mapping
映射节点的常量折叠规则：
- 缩放为零 -> 输出零
- 恒等变换（无平移、无旋转、缩放为一，且类型非 NORMAL）-> 绕过输入

## 依赖关系

- **内部头文件**:
  - `kernel/svm/types.h` — SVM 节点类型（NodeMathType、NodeMix 等）
  - `util/types.h` — 基本类型
- **实现文件额外依赖**:
  - `scene/shader_graph.h` — 着色器图（用于 connect/disconnect 操作）
  - `util/log.h` — 日志输出（LOG_TRACE 级别记录折叠操作）
- **被引用**: `scene/shader_graph.cpp`（在 `ShaderGraph::constant_fold()` 中创建 ConstantFolder 实例）、`scene/shader_nodes.cpp`（各节点的 `constant_fold()` 方法接收 ConstantFolder 引用）

## 实现细节 / 关键算法

### 折叠执行流程
1. `ShaderGraph::constant_fold()` 按拓扑序遍历所有节点
2. 对每个节点的每个有下游连接的输出，创建 `ConstantFolder` 实例
3. 调用 `node->constant_fold(folder)`，由节点自身决定如何折叠
4. 折叠操作直接修改着色器图（断开连接、写入常量值、重新链接）
5. 若折叠移除了位移输出的所有连接，自动插入 ColorNode 保持图有效性

### 常量标记机制
当输入被常量折叠时，设置 `input->constant_folded_in = true`。这使得 SVM 编译器的 `is_linked()` 检查能区分"真正未连接"和"被常量折叠后断开"的输入，避免过度优化（例如不会为已折叠的输入再次加载默认值）。

### 绕过安全性
`bypass()` 方法不使用 `graph->relink()` 是因为后者会影响节点输入，在节点有多个输出且被多次折叠时不安全。改为手动断开原始连接并逐一重新连接到新输出。

### 钳制(Clamp)处理
当启用钳制时：
- 如果可以用常量替换，先钳制常量值再替换
- 如果需要绕过，不能直接绕过（因为绕过后没有钳制节点），此时断开其他输入以简化但保留节点
- `try_bypass_or_make_constant()` 在无法完全折叠时返回 false

## 关联文件

- `src/scene/shader_graph.h` / `shader_graph.cpp` — 着色器图（调用常量折叠）
- `src/scene/shader_nodes.h` / `shader_nodes.cpp` — 节点类型（实现各自的 constant_fold 方法）
- `src/scene/svm.h` / `svm.cpp` — SVM 编译器（受益于常量折叠减少的指令数）
