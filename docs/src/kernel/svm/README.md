# kernel/svm - 着色器虚拟机 (SVM)

## 概述

`kernel/svm` 模块实现了 Cycles 路径追踪渲染器的着色器虚拟机（Shader Virtual Machine）。SVM 是一个基于栈的字节码解释器，执行由着色器编译器生成的节点序列，计算表面、体积和位移着色的最终结果。

着色器被编译为 `uint4` 指令序列存储在 1D 纹理中。SVM 解释器按顺序逐节点执行，通过固定大小的浮点栈（255 个元素）传递中间结果。最终输出是一个或多个闭包（Closure），包含闭包类型、权重和参数数据。混合闭包（Mix Closure）的逻辑主要在 SVM 编译器中处理。

本目录包含约 57 个文件，按着色器节点类型可分为纹理节点、颜色节点、数学节点、向量节点、闭包节点、转换节点和其他节点七大类。

## 目录结构

### 核心框架文件

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `svm.h` | 主解释器 | SVM 主循环：`svm_eval_nodes()` 模板函数，巨型 switch-case 分发所有节点类型 |
| `types.h` | 类型定义 | SVM 栈常量（`SVM_STACK_SIZE=255`）、`ShaderNodeType` 枚举（通过模板生成）、各类节点参数枚举 |
| `node_types_template.h` | 节点注册 | 所有 SVM 节点类型的模板注册宏 |
| `util.h` | 工具函数 | 栈操作（`stack_load_float`、`stack_store_float3`）、节点读取（`read_node`）、数据打包/解包 |

### 纹理节点（程序化纹理）

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `noisetex.h` | 噪声纹理 | Perlin 噪声纹理节点，支持 1D-4D 维度，输出 fBm 和 distorted noise |
| `voronoi.h` | Voronoi 纹理 | Voronoi/Worley 噪声纹理，支持多种距离度量和特征输出（F1、F2、平滑 F1 等） |
| `gabor.h` | Gabor 纹理 | Gabor 噪声纹理节点，基于随机 Gabor 核的各向异性/各向同性程序化纹理 |
| `wave.h` | 波纹纹理 | 条带/环形波纹纹理，支持多种波形和失真 |
| `magic.h` | 魔法纹理 | 基于三角函数组合的程序化颜色纹理 |
| `checker.h` | 棋盘纹理 | 棋盘格纹理，交替两种颜色 |
| `brick.h` | 砖墙纹理 | 砖墙图案纹理，支持砖块大小、间距、偏移等参数 |
| `gradient.h` | 渐变纹理 | 线性/二次/球形/径向等多种渐变模式 |
| `sky.h` | 天空纹理 | 物理天空模型纹理（Preetham/Hosek-Wilkie/Nishita 模型） |
| `image.h` | 图像纹理 | 图像纹理节点，从纹理图像采样，支持盒投影（box mapping） |
| `ies.h` | IES 纹理 | IES 配光曲线纹理，用于真实灯具光分布 |
| `white_noise.h` | 白噪声 | 基于哈希的白噪声纹理，支持 1D-4D 输入 |
| `fractal_noise.h` | 分形噪声 | fBm 分形噪声工具函数（被噪声纹理和其他纹理节点引用） |
| `noise.h` | 噪声基础 | Perlin 噪声基础函数（被 `noisetex.h` 引用，非独立节点） |
| `radial_tiling.h` | 径向平铺 | 径向平铺纹理坐标变换节点 |
| `radial_tiling_shared.h` | 径向平铺共享 | 径向平铺的共享数学函数 |

### 颜色节点

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `mix.h` | 混合 | 颜色混合节点，支持 18 种混合模式（叠加、屏幕、相乘、差值等）；也包含浮点和向量混合 |
| `brightness.h` | 亮度/对比度 | 亮度对比度调整节点 |
| `gamma.h` | Gamma | Gamma 校正节点 |
| `hsv.h` | HSV | 色相/饱和度/明度调整节点 |
| `invert.h` | 反转 | 颜色反转节点 |
| `ramp.h` | 颜色渐变 | RGB 渐变/曲线查找表节点，支持 RGB 曲线和颜色渐变 |
| `ramp_util.h` | 渐变工具 | 渐变查找和插值工具函数 |
| `color_util.h` | 颜色工具 | 颜色空间转换工具函数（HSV-RGB 等） |
| `blackbody.h` | 黑体辐射 | 基于温度的黑体辐射颜色转换 |
| `wavelength.h` | 波长 | 可见光波长到 RGB 颜色的转换 |
| `sepcomb_color.h` | 分离/合并颜色 | 颜色分量分离与合并（RGB/HSV/HSL） |

### 数学节点

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `math.h` | 数学运算 | 标量数学运算节点（加、减、乘、除、幂、对数、三角函数等 30+ 种运算） |
| `math_util.h` | 数学工具 | 数学节点的共享工具函数 |
| `clamp.h` | 钳制 | 数值钳制节点（min-max 和 range 模式） |
| `map_range.h` | 映射范围 | 数值区间映射节点，支持线性/阶梯/平滑阶梯等插值模式；也支持向量版本 |

### 向量节点

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `normal.h` | 法线 | 法线节点，输出法线和点积 |
| `mapping.h` | 映射 | 向量映射节点（点/纹理/向量/法线模式的变换） |
| `mapping_util.h` | 映射工具 | 纹理坐标映射变换工具函数 |
| `vector_rotate.h` | 向量旋转 | 向量绕轴/欧拉角旋转节点 |
| `vector_transform.h` | 向量变换 | 向量空间变换节点（世界/对象/相机空间之间） |
| `sepcomb_vector.h` | 分离/合并向量 | 向量分量分离与合并 |
| `bump.h` | 凹凸 | 凹凸贴图节点，通过高度差异计算扰动法线 |
| `displace.h` | 位移 | 位移节点，将高度/向量位移应用到表面 |

### 闭包节点（BSDF/发光/体积）

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `closure.h` | 闭包核心 | 所有 BSDF/BSSRDF/发射/体积闭包的创建和参数设置，包含 Principled BSDF 的完整实现 |
| `fresnel.h` | 菲涅尔 | 菲涅尔反射率和层权重（Layer Weight）节点 |
| `ao.h` | 环境光遮蔽 | AO 节点，需要 `__SHADER_RAYTRACE__` 特性（实际发射光线） |
| `bevel.h` | 倒角 | 光线追踪倒角节点，需要 `__SHADER_RAYTRACE__` 特性 |

### 转换节点

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `convert.h` | 类型转换 | 浮点/整数/颜色/向量之间的类型转换节点 |
| `value.h` | 常量值 | 浮点和向量常量值节点 |

### 输入/信息节点

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `geometry.h` | 几何信息 | 几何数据输入节点：位置 P、法线 N/Ng、切线 T、入射方向 I、UV 等 |
| `camera.h` | 相机信息 | 相机数据节点：视线深度、视线距离 |
| `light_path.h` | 光线路径 | 光线路径信息节点：光线类型查询（相机/阴影/漫射/光泽等）、路径深度和长度 |
| `attribute.h` | 属性 | 通用几何属性读取节点（UV、颜色属性等） |
| `vertex_color.h` | 顶点颜色 | 顶点颜色属性读取节点 |
| `tex_coord.h` | 纹理坐标 | 纹理坐标节点：生成坐标、法线、UV、对象空间、相机空间、窗口/反射坐标 |
| `wireframe.h` | 线框 | 线框距离计算节点 |
| `aov.h` | AOV 输出 | 任意输出变量（AOV）写入节点 |

## 核心类与数据结构

- **SVM 栈**：固定大小 `float stack[SVM_STACK_SIZE]`（255 元素）的局部数组，存储所有中间值（标量、颜色、向量均以 float 存储）。GPU 上存储在局部内存中。
- **`ShaderNodeType` 枚举**：通过 `node_types_template.h` 模板生成的所有节点类型枚举（`NODE_END`、`NODE_CLOSURE_BSDF`、`NODE_TEX_NOISE` 等）。
- **`closure_weight`**：当前闭包权重（`Spectrum` 类型），由 `NODE_CLOSURE_SET_WEIGHT`/`NODE_CLOSURE_WEIGHT` 设置，传递给后续闭包创建节点。
- **`SVM_STACK_INVALID` (255)**：标记无效的栈偏移，表示参数不在栈上。

## 内核函数入口

- **`svm_eval_nodes<node_feature_mask, type, State>()`**：SVM 主解释器循环。模板参数 `node_feature_mask` 控制编译时特性剪裁（BUMP、EMISSION、VOLUME 等），`type` 区分表面/体积/位移着色类型。
- **`NODE_SHADER_JUMP`**：根据着色器类型（表面/体积/位移）跳转到对应的节点序列起始位置。
- **`NODE_END`**：终止 SVM 执行。

## GPU 兼容性

SVM 对 GPU 进行了深度优化：

- **栈在局部内存中**：255 个 float 的栈存储在 GPU 局部内存（local memory）而非寄存器，因为索引方式在编译时未知。同一着色器的并行执行具有内存访问合并和缓存优势。
- **编译时特性剪裁**：`__KERNEL_USE_DATA_CONSTANTS__` 启用时，`SVM_CASE` 宏通过运行时常量数据跳过未使用的节点类型，减少指令缓存压力。
- **节点特性掩码**：`node_feature_mask` 模板参数允许编译器在编译时消除不需要的节点代码路径（如纯表面着色时消除体积节点代码）。
- **条件编译**：`__SHADER_RAYTRACE__`（AO/Bevel）、`__HAIR__`、`__POINTCLOUD__` 等特性宏控制可选节点的编译。
- **`ccl_device_noinline` / `ccl_device_noinline_cpu`**：部分大型节点函数（如 `svm_node_closure_bsdf`）标记为不内联，减少 GPU 内核的寄存器压力。

## 依赖关系

### 上游依赖（本模块依赖）
- `kernel/globals.h`：全局内核数据访问
- `kernel/types.h`：基础类型定义
- `kernel/closure/`：所有 BSDF/BSSRDF/发射/体积闭包的分配和评估函数
- `kernel/geom/`：几何体属性读取（`primitive_surface_attribute` 等）、对象元数据
- `kernel/camera/`：相机参数（用于纹理坐标计算）
- `kernel/util/colorspace.h`：色彩空间转换

### 下游依赖（依赖本模块）
- `kernel/integrator/surface_shader.h`：调用 `svm_eval_nodes<SURFACE>()` 评估表面着色器
- `kernel/integrator/volume_shader.h`：调用 `svm_eval_nodes<VOLUME>()` 评估体积着色器
- `kernel/integrator/displacement_shader.h`：调用 `svm_eval_nodes<DISPLACEMENT>()` 评估位移着色器
- `kernel/light/sample.h`：评估光源着色器获取发射光谱

## 关键算法与实现细节

1. **字节码解释器架构**：SVM 使用 `uint4` 为基本指令单元。每条指令的 `x` 分量为节点类型，`y`/`z`/`w` 为压缩参数。大型节点（如 Principled BSDF、Voronoi）通过递增偏移量读取额外数据节点。浮点值通过 `__int_as_float` 编码在 uint 中。

2. **闭包混合优化**：`NODE_MIX_CLOSURE` 配合 `NODE_JUMP_IF_ZERO`/`NODE_JUMP_IF_ONE` 实现高效的闭包混合。当混合因子为 0 或 1 时，跳过不需要的闭包分支评估，避免不必要的计算。

3. **凹凸贴图实现**：凹凸节点使用三次采样策略 -- 在原始点及其 dx/dy 微分偏移处分别评估高度值。`NODE_ENTER_BUMP_EVAL`/`NODE_LEAVE_BUMP_EVAL` 保存和恢复 `ShaderData` 中的位置信息，`NODE_GEOMETRY_BUMP_DX`/`NODE_GEOMETRY_BUMP_DY` 提供偏移后的几何数据。

4. **Principled BSDF**：`closure.h` 中的 `svm_node_closure_bsdf()` 实现了完整的 Principled BSDF 着色模型，在单一节点中组合漫反射、次表面散射、光泽反射、清漆层、透射等多个闭包层。

5. **纹理节点性能**：程序化纹理节点（Noise、Voronoi、Gabor 等）包含大量数学运算。这些节点被设计为纯函数式（无副作用），有利于 GPU 并行执行。Voronoi 纹理使用 `node_feature_mask` 模板参数在编译时裁剪未使用的距离度量和特征输出。

6. **光线追踪节点**：AO 和 Bevel 节点（`__SHADER_RAYTRACE__`）是特殊节点 -- 它们在着色器评估过程中发射额外的光线。这需要在 GPU 波前调度中进行特殊处理，使用单独的 `shade_surface_raytrace` 内核执行包含这些节点的着色器。

## 参见

- `src/kernel/closure/` - BSDF/BSSRDF 闭包实现
- `src/render/svm.cpp` - 主机端 SVM 编译器（将节点图编译为字节码）
- `src/render/nodes.cpp` - 着色器节点定义和参数
- `src/kernel/integrator/surface_shader.h` - SVM 执行的调用方
