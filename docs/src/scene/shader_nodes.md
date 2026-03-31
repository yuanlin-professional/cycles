# shader_nodes.h / shader_nodes.cpp - 着色器节点类型定义

## 概述

本文件定义了 Cycles 渲染器中所有具体的着色器节点类型，是着色器系统中规模最大的文件。涵盖了纹理节点、双向散射分布函数(BSDF)节点、体积节点、输入节点、颜色处理节点、向量运算节点、闭包混合节点等完整的着色器节点库。每个节点类均实现了 `compile(SVMCompiler&)` 和 `compile(OSLCompiler&)` 两种编译后端，以及可选的 `constant_fold()` 常量折叠优化。

## 类与结构体

### 纹理映射与基类

#### TextureMapping
- **功能**: 纹理坐标映射配置（平移、旋转、缩放、轴映射、投影方式）
- **关键成员**: `translation`, `rotation`, `scale`, `type`(POINT/TEXTURE/VECTOR/NORMAL), `projection`(FLAT/CUBE/TUBE/SPHERE), `x/y/z_mapping`
- **关键方法**:
  - `compute_transform()` — 计算最终变换矩阵
  - `skip()` — 检查是否为恒等变换（可跳过）
  - `compile()` — 编译到 SVM/OSL

#### TextureNode
- **继承**: `ShaderNode`
- **功能**: 所有纹理节点的基类，包含 TextureMapping

#### ImageSlotTextureNode
- **继承**: `TextureNode`
- **功能**: 使用图像管理器(ImageManager)插槽的纹理节点基类

### 纹理节点

| 节点类 | 功能 |
|--------|------|
| `ImageTextureNode` | 图像纹理（支持 UDIM 瓦片、动画、多种投影和插值） |
| `EnvironmentTextureNode` | 环境贴图纹理 |
| `SkyTextureNode` | 程序化天空纹理（Hosek / Nishita 模型） |
| `GradientTextureNode` | 渐变纹理 |
| `NoiseTextureNode` | 噪声纹理（支持多维度和多种类型） |
| `GaborTextureNode` | Gabor 纹理 |
| `VoronoiTextureNode` | Voronoi 纹理 |
| `WaveTextureNode` | 波浪纹理 |
| `MagicTextureNode` | 魔法纹理 |
| `CheckerTextureNode` | 棋盘格纹理 |
| `BrickTextureNode` | 砖块纹理 |
| `IESLightNode` | IES 灯光配光曲线 |
| `WhiteNoiseTextureNode` | 白噪声 |

### 输出节点

#### OutputNode
- **继承**: `ShaderNode`
- **功能**: 着色器图的最终输出节点，包含 Surface、Volume、Displacement、Normal 四个输入插口
- **特殊**: 不允许去重（`equals()` 始终返回 false）

#### OutputAOVNode
- **功能**: AOV（Arbitrary Output Variable）输出节点，将值/颜色写入自定义渲染通道

### BSDF 节点（表面着色）

#### BsdfBaseNode
- **继承**: `ShaderNode`
- **功能**: 所有双向散射分布函数(BSDF)节点的基类，声明闭包类型和线性运算特性
- **关键成员**: `closure` — 闭包类型

#### BsdfNode
- **继承**: `BsdfBaseNode`
- **功能**: 标准 BSDF 节点基类，包含 color、normal、surface_mix_weight 输入

#### 具体 BSDF 节点

| 节点类 | 功能 |
|--------|------|
| `DiffuseBsdfNode` | 漫反射 BSDF |
| `PrincipledBsdfNode` | 原理化 BSDF（Disney BRDF 变体），支持金属、玻璃、次表面散射、光泽、涂层、薄膜干涉等 |
| `TranslucentBsdfNode` | 半透明 BSDF |
| `TransparentBsdfNode` | 透明 BSDF |
| `RayPortalBsdfNode` | 光线传送门 BSDF |
| `SheenBsdfNode` | 光泽 BSDF |
| `MetallicBsdfNode` | 金属 BSDF（支持物理导体菲涅耳、F82 色调） |
| `GlossyBsdfNode` | 光泽反射 BSDF |
| `GlassBsdfNode` | 玻璃 BSDF |
| `RefractionBsdfNode` | 折射 BSDF |
| `ToonBsdfNode` | 卡通着色 BSDF |
| `SubsurfaceScatteringNode` | 次表面散射节点 |
| `PrincipledHairBsdfNode` | 原理化毛发 BSDF（支持 Chiang/Huang 模型） |
| `HairBsdfNode` | 毛发 BSDF |

### 体积节点

| 节点类 | 功能 |
|--------|------|
| `VolumeNode` | 体积节点基类 |
| `AbsorptionVolumeNode` | 吸收体积 |
| `ScatterVolumeNode` | 散射体积 |
| `VolumeCoefficientsNode` | 体积系数节点 |
| `PrincipledVolumeNode` | 原理化体积（支持密度/颜色/温度属性） |

### 发射与背景节点

| 节点类 | 功能 |
|--------|------|
| `EmissionNode` | 发射节点（表面和体积共用） |
| `BackgroundNode` | 背景节点 |
| `HoldoutNode` | 遮罩节点 |
| `AmbientOcclusionNode` | 环境遮蔽节点（需光线追踪） |

### 闭包操作节点

| 节点类 | 功能 |
|--------|------|
| `AddClosureNode` | 闭包相加 |
| `MixClosureNode` | 闭包混合（带因子） |
| `MixClosureWeightNode` | 闭包混合权重（编译器内部使用） |

### 输入/信息节点

| 节点类 | 功能 |
|--------|------|
| `GeometryNode` | 几何信息（位置、法线、切线等） |
| `TextureCoordinateNode` | 纹理坐标 |
| `UVMapNode` | UV 贴图 |
| `LightPathNode` | 光线路径信息 |
| `LightFalloffNode` | 灯光衰减 |
| `ObjectInfoNode` | 对象信息 |
| `ParticleInfoNode` | 粒子信息 |
| `HairInfoNode` | 毛发信息 |
| `PointInfoNode` | 点云信息 |
| `VolumeInfoNode` | 体积信息 |
| `VertexColorNode` | 顶点颜色 |
| `AttributeNode` | 自定义属性 |
| `CameraNode` | 摄像机信息 |
| `FresnelNode` | 菲涅耳效果 |
| `LayerWeightNode` | 层权重 |
| `WireframeNode` | 线框 |

### 颜色与数学节点

| 节点类 | 功能 |
|--------|------|
| `MathNode` | 标量数学运算 |
| `VectorMathNode` | 向量数学运算 |
| `MixNode` / `MixColorNode` / `MixFloatNode` / `MixVectorNode` | 混合节点系列 |
| `ColorNode` / `ValueNode` | 常量颜色/值节点 |
| `RGBToBWNode` | RGB 转灰度 |
| `ConvertNode` | 类型转换节点 |
| `MappingNode` | 向量映射（点/纹理/向量/法线） |
| `CombineColorNode` / `SeparateColorNode` | 颜色通道组合/分离 |
| `CombineXYZNode` / `SeparateXYZNode` | XYZ 分量组合/分离 |
| `GammaNode` | 伽马校正 |
| `BrightContrastNode` | 亮度对比度 |
| `HSVNode` | HSV 色彩调整 |
| `InvertNode` | 反转 |
| `BlackbodyNode` | 黑体辐射 |
| `WavelengthNode` | 波长转颜色 |
| `ClampNode` | 钳制 |
| `MapRangeNode` / `VectorMapRangeNode` | 范围映射 |

### 曲线节点

| 节点类 | 功能 |
|--------|------|
| `CurvesNode` | 曲线节点基类 |
| `RGBCurvesNode` | RGB 曲线 |
| `VectorCurvesNode` | 向量曲线 |
| `FloatCurveNode` | 浮点曲线 |
| `RGBRampNode` | RGB 渐变 |

### 法线与位移节点

| 节点类 | 功能 |
|--------|------|
| `NormalNode` | 法线节点 |
| `NormalMapNode` | 法线贴图 |
| `TangentNode` | 切线节点 |
| `BumpNode` | 凹凸节点 |
| `SetNormalNode` | 设置法线（内部用于位移凹凸） |
| `DisplacementNode` | 标量位移 |
| `VectorDisplacementNode` | 向量位移 |

### 其他

| 节点类 | 功能 |
|--------|------|
| `VectorRotateNode` | 向量旋转 |
| `VectorTransformNode` | 向量空间变换 |
| `BevelNode` | 倒角法线（需光线追踪） |
| `RadialTilingNode` | 径向平铺 |
| `OSLNode` | 开放着色语言(OSL)自定义着色器节点 |

## 核心函数

每个节点类都实现以下编译方法：
- `compile(SVMCompiler&)` — 编译为着色器虚拟机(SVM)指令
- `compile(OSLCompiler&)` — 编译为开放着色语言(OSL)着色器调用

许多节点还实现：
- `constant_fold(const ConstantFolder&)` — 常量折叠优化
- `expand(ShaderGraph*)` — 节点展开为更基础的子图
- `simplify_settings(Scene*)` — 运行时简化设置

## 依赖关系

- **内部头文件**:
  - `graph/node.h` — 节点基类
  - `kernel/svm/types.h` — SVM 节点类型定义
  - `scene/image.h` — 图像句柄
  - `scene/shader_graph.h` — 着色器图基类
  - `util/array.h`, `util/string.h`, `util/unique_ptr.h`
- **实现文件额外依赖**:
  - `scene/constant_fold.h`, `scene/colorspace.h`, `scene/film.h`, `scene/image_sky.h`
  - `scene/integrator.h`, `scene/light.h`, `scene/mesh.h`, `scene/osl.h`, `scene/scene.h`, `scene/svm.h`
  - `sky_hosek.h`, `sky_nishita.h` — 天空模型
  - `kernel/svm/color_util.h`, `kernel/svm/mapping_util.h`, `kernel/svm/math_util.h`, `kernel/svm/ramp_util.h`
- **被引用**: `scene/shader.cpp`, `scene/shader_graph.cpp`, `scene/svm.cpp`, `scene/osl.cpp`, `scene/osl.h`, `scene/light.cpp`, `scene/geometry*.cpp`, `scene/background.cpp`, `test/render_graph_finalize_test.cpp`, `hydra/*.cpp`, `app/cycles_xml.cpp` 等 17 个文件

## 实现细节 / 关键算法

### PrincipledBsdfNode
原理化 BSDF 节点是最复杂的着色器节点，支持多达 12 个闭包。包含金属、玻璃、次表面散射、光泽、涂层、薄膜干涉等多种材质层。`simplify_settings()` 会在编译前优化不需要的组件（如断开未使用的输入连接）。

### TextureMapping::compute_transform()
根据映射类型计算不同的变换矩阵：
- TEXTURE: 逆变换（纹理坐标的逆变换等价于纹理的正变换）
- POINT: 完整 TRS 变换
- VECTOR: 无平移
- NORMAL: 逆转置矩阵

### ConvertNode
类型转换节点使用静态数组 `node_types[MAX_TYPE][MAX_TYPE]` 为所有可能的类型对注册专门的节点类型，避免运行时动态分配。

### OSLNode
开放着色语言(OSL)节点使用自定义内存分配（placement new），在节点末尾附加额外存储空间用于动态输入的默认值。

## 关联文件

- `src/scene/shader_graph.h` / `shader_graph.cpp` — 着色器图基类（ShaderNode 的容器）
- `src/scene/shader.h` / `shader.cpp` — 着色器管理
- `src/scene/svm.h` / `svm.cpp` — SVM 编译器（调用各节点的 compile）
- `src/scene/osl.h` / `osl.cpp` — OSL 编译器
- `src/scene/constant_fold.h` / `constant_fold.cpp` — 常量折叠辅助类
- `src/scene/image.h` — 图像纹理管理
