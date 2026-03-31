# svm.h / svm.cpp - 着色器虚拟机(SVM)管理器与编译器

## 概述

本文件实现了 Cycles 渲染器的着色器虚拟机(SVM)编译系统，是默认的着色器编译后端（另一个是开放着色语言(OSL)后端）。包含两个核心类：`SVMShaderManager` 作为 SVM 后端的着色器管理器，负责多线程并行编译所有着色器并将字节码上传到设备；`SVMCompiler` 负责将着色器图(ShaderGraph)编译为 SVM 字节码指令序列（int4 数组），管理虚拟栈分配和多重闭包编译。

## 类与结构体

### SVMShaderManager

- **继承**: `ShaderManager`（来自 `scene/shader.h`）
- **功能**: SVM 后端的着色器管理器，负责着色器的并行编译和设备数据管理
- **关键方法**:
  - `device_update_specific()` — SVM 特有的设备更新：使用 TaskPool 并行编译所有着色器，构建全局跳转表，将 SVM 字节码拷贝到设备。全局节点列表结构为"跳转表（每个着色器一个条目）+ 所有着色器的节点序列"
  - `device_free()` — 释放设备上的 SVM 节点内存
  - `device_update_shader()` — 编译单个着色器：创建 SVMCompiler，调用其 compile 方法

### SVMCompiler

- **继承**: 无
- **功能**: 将着色器图编译为 SVM 字节码。管理虚拟栈的分配和回收，处理闭包树的遍历和编译，支持多重闭包模式下的条件跳转优化。
- **关键成员**:
  - `scene` — 场景指针
  - `current_graph` — 当前编译的着色器图
  - `current_svm_nodes` — 当前编译产生的 SVM 指令序列（array<int4>）
  - `current_type` — 当前编译的着色器类型（Surface/Volume/Displacement/Bump）
  - `current_shader` — 当前编译的着色器
  - `active_stack` — 活动栈状态（用户引用计数数组）
  - `max_stack_use` — 峰值栈使用量
  - `mix_weight_offset` — 闭包混合权重的栈偏移
  - `bump_state_offset` — 凹凸状态的栈偏移
  - `compile_failed` — 编译是否因栈溢出失败
  - `svm_node_types_used` — 跟踪使用了哪些 SVM 节点类型（用于内核裁剪）
  - `background` — 是否为背景着色器
- **关键方法**:
  - `compile(Shader*, svm_nodes, index, summary)` — 编译完整着色器：按顺序编译 Bump -> Surface -> Volume -> Displacement 四个类型，合并到全局节点数组，最后调用 `estimate_emission()` 估算发射
  - `compile_type(Shader*, graph, type)` — 编译特定着色器类型
  - `stack_assign(ShaderInput*)` — 为输入分配栈空间（已链接则共享输出栈位置，未链接则加载默认值）
  - `stack_assign(ShaderOutput*)` — 为输出分配栈空间
  - `stack_assign_if_linked()` — 仅在链接时分配栈
  - `stack_assign_if_not_equal()` — 仅在值不等于指定默认值时分配栈
  - `stack_find_offset(size/type)` — 在栈中寻找连续空闲空间
  - `stack_clear_offset()` — 释放栈空间
  - `stack_link()` — 链接输入输出共享栈位置
  - `add_node(...)` — 添加 SVM 指令节点（多种重载）
  - `attribute(name/std)` — 获取属性 ID
  - `encode_uchar4()` — 将 4 个 uint8 编码到单个 uint32
  - `is_linked()` — 检查输入是否已链接或被常量折叠

### SVMCompiler::Summary

- **功能**: 编译摘要信息，用于调试和性能分析
- **关键成员**:
  - `num_svm_nodes` — 编译后的 SVM 节点数量
  - `peak_stack_usage` — 峰值栈使用量
  - `time_generate_surface` / `time_generate_bump` / `time_generate_volume` / `time_generate_displacement` — 各类型着色器的编译时间
  - `time_total` — 总编译时间
- **关键方法**: `full_report()` — 返回完整的多行编译报告字符串

### SVMCompiler::Stack

- **功能**: 模拟虚拟栈，跟踪每个栈位置的使用者数量
- **关键成员**: `users[SVM_STACK_SIZE]` — 每个栈位置的引用计数

### SVMCompiler::CompilerState

- **功能**: 编译器的全局状态，贯穿整个编译过程
- **关键成员**:
  - `nodes_done` — 已编译的节点集合（ShaderNodeSet）
  - `closure_done` — 已编译的闭包节点集合
  - `aov_nodes` — AOV 输出节点集合
  - `nodes_done_flag` — 已编译标志数组（按节点 ID 索引）
  - `node_feature_mask` — 当前允许编译的节点特性掩码

## 核心函数

### 编译流程

1. **compile()** — 入口函数：
   - 插入跳转节点作为着色器入口
   - 依次调用 `compile_type()` 编译 Bump（如有位移凹凸）、Surface、Volume、Displacement
   - 将各类型的跳转偏移写入入口跳转节点
   - 记录编译统计

2. **compile_type()** — 单类型编译：
   - 重置栈和节点状态
   - 若为 Bump 类型且使用 DISPLACE_BOTH，插入 NODE_ENTER_BUMP_EVAL 保存凹凸状态
   - 设置节点特性掩码（不同着色器类型允许不同的节点特性）
   - 调用 `generate_multi_closure()` 遍历闭包树
   - 编译输出节点
   - 处理 AOV 节点（插入 NODE_AOV_START）
   - 若为 Bump 类型，插入 NODE_LEAVE_BUMP_EVAL 恢复状态
   - 非 Bump 类型以 NODE_END 结束

3. **generate_multi_closure()** — 多重闭包编译：
   - 对组合闭包节点(COMBINE_CLOSURE)：
     - 编译混合因子的依赖节点
     - 分析两个输入闭包的共享依赖和独有依赖
     - 先编译共享依赖
     - 对每个输入闭包插入条件跳转（NODE_JUMP_IF_ONE / NODE_JUMP_IF_ZERO），跳过权重为零的分支
   - 对叶子闭包节点：调用 `generate_closure_node()`

4. **generate_closure_node()** — 单闭包编译：
   - 检查节点特性是否匹配当前着色器类型
   - 编译闭包的所有依赖节点
   - 设置闭包混合权重
   - 编译闭包本身（调用 `node->compile(*this)`）
   - 更新着色器标志（透明、BSSRDF、凹凸等）

### 并行编译架构

`device_update_specific()` 使用 TaskPool 并行编译所有着色器：
1. 为每个着色器分配独立的 `array<int4>` 存储编译结果
2. 每个编译任务创建独立的 SVMCompiler 实例
3. 所有着色器编译完成后，汇总为全局节点数组
4. 全局数组结构：`[跳转表: N 个 int4] [着色器0节点...] [着色器1节点...] ...`
5. 调整跳转表偏移为全局偏移

## 依赖关系

- **内部头文件**:
  - `scene/shader.h` — ShaderManager 基类
  - `scene/shader_graph.h` — 着色器图
  - `util/array.h`, `util/string.h`
- **实现文件额外依赖**:
  - `device/device.h` — 设备接口
  - `scene/background.h`, `scene/light.h`, `scene/mesh.h`, `scene/scene.h`, `scene/shader_nodes.h`, `scene/stats.h`
  - `util/log.h`, `util/progress.h`, `util/task.h`
- **被引用**: `scene/shader.cpp`, `scene/shader_nodes.cpp`, `scene/svm.cpp`, `scene/scene.cpp`

## 实现细节 / 关键算法

### 虚拟栈管理
SVM 使用固定大小的栈（`SVM_STACK_SIZE`）。编译器通过引用计数跟踪每个栈位置的使用者：
- `stack_find_offset()` — 寻找 N 个连续空闲位置，标记为已使用
- `stack_clear_offset()` — 减少引用计数
- `stack_clear_users()` — 当某个节点的所有消费者都已编译完成时，释放其输出占用的栈空间
- `stack_clear_temporary()` — 清理临时加载的默认值栈空间
- 栈溢出时设置 `compile_failed`，生成空着色器

### 条件跳转优化
在多重闭包模式下，对 MixClosure 节点：
- 若混合因子可链接，插入 NODE_JUMP_IF_ONE 跳过 Closure1，NODE_JUMP_IF_ZERO 跳过 Closure2
- 跳转距离在编译后回填（因为编译时不知道闭包子树的大小）
- 对 AddClosure 或固定权重的 MixClosure，不插入跳转

### 共享依赖处理
`generate_multi_closure()` 在处理 MixClosure 时：
1. 分析两个闭包输入的依赖集合
2. 计算交集作为共享依赖
3. 还检查父节点的依赖，确保被父节点使用的节点不会被跳过
4. 检查 AOV 依赖，确保 AOV 节点不会被错误跳过
5. 先编译共享依赖，再分别编译独有依赖

### AOV 支持
- AOV 节点仅在对象直接可见时写入
- 编译器在闭包编译完成后插入 NODE_AOV_START 标记
- 内核可在该标记处提前终止（若 AOV 不需要写入）

## 关联文件

- `src/scene/shader.h` / `shader.cpp` — ShaderManager 基类，device_update_pre/post
- `src/scene/shader_graph.h` / `shader_graph.cpp` — 着色器图（编译输入）
- `src/scene/shader_nodes.h` / `shader_nodes.cpp` — 节点类型（每个节点有 SVM compile 方法）
- `src/scene/osl.h` / `osl.cpp` — OSL 编译器（替代后端）
- `src/scene/constant_fold.h` — 常量折叠（编译前优化）
