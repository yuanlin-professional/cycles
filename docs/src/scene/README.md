# Scene 模块 - Cycles 场景描述与管理

## 概述

`src/scene/` 是 Cycles 路径追踪渲染器的**场景描述层**，负责定义渲染所需的全部数据：几何体、材质、光源、相机、图像纹理以及积分器参数。该模块在宿主应用（如 Blender）与底层内核（kernel）之间充当桥梁：

1. 宿主应用通过 `Scene` 对象的 API 构建和修改场景元素；
2. 各 Manager 类检测变更标志（dirty flags），将增量数据上传至设备（CPU/GPU）；
3. 内核使用 `DeviceScene` 中的扁平化设备向量执行路径追踪。

模块核心设计理念：
- **节点系统（Node System）**：所有场景元素均继承自 `graph/node.h` 中的 `Node`，通过 `NODE_SOCKET_API` 宏暴露属性，实现统一的序列化、变更跟踪和参数绑定。
- **Manager 模式**：每类场景元素配有对应的 Manager（如 `GeometryManager`、`LightManager`、`ShaderManager`），Manager 持有 `update_flags` 位掩码，仅在标记变更时执行 `device_update`。
- **所有权管理**：`Scene` 通过 `unique_ptr_vector` 拥有所有场景节点，支持 `create_node<T>()` / `delete_node()` 模板接口。

## 目录结构

### 几何体（Geometry）

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `geometry.h/cpp` | 核心 | 几何体抽象基类 `Geometry`（继承自 `Node`），定义 Mesh/Hair/Volume/PointCloud/Light 五种类型枚举，管理属性集、BVH 构建、运动模糊步数；`GeometryManager` 负责设备端全局几何数据上传 |
| `geometry_attributes.cpp` | 实现 | 几何体属性（SVM/OSL 属性映射）的设备更新逻辑 |
| `geometry_bvh.cpp` | 实现 | 每个几何体的 BVH 构建与更新，包括顶层 BVH 组装 |
| `geometry_mesh.cpp` | 实现 | 三角网格和曲线数据的设备打包与上传 |
| `mesh.h/cpp` | 核心 | `Mesh` 类（继承 `Geometry`），存储三角面、顶点、法线、细分曲面（Catmull-Clark / Linear）参数；支持 UDIM 瓦片；内含 `Triangle`、`SubdFace`、`SubdEdgeCrease` 结构体 |
| `mesh_displace.cpp` | 实现 | 真位移贴图（True Displacement）：在设备上评估位移着色器并回写顶点位置 |
| `mesh_subdivision.cpp` | 实现 | 细分曲面的棋盘化（tessellation），调用 `subd/` 模块的 `DiagSplit` |
| `hair.h/cpp` | 核心 | `Hair` 类（继承 `Geometry`），存储曲线关键点（`curve_keys`）、半径、运动模糊数据；支持 Ribbon 和 Round 两种曲线形状 |
| `curves.cpp` | 实现 | 曲线几何的额外处理（与 `hair.cpp` 配合） |
| `volume.h/cpp` | 核心 | `Volume` 类（继承 `Mesh`），增加 `step_size`、`object_space`、`velocity_scale` 属性；`VolumeManager` 管理体积八叉树（Octree）构建、步长更新、OpenVDB SDF 网格 |
| `pointcloud.h/cpp` | 核心 | `PointCloud` 类（继承 `Geometry`），存储点位置、半径、着色器索引；支持运动模糊 |
| `object.h/cpp` | 核心 | `Object` 类（继承 `Node`），表示场景中的几何实例——持有指向 `Geometry` 的引用及变换矩阵、可见性、运动模糊、阴影捕获器、光链接等属性；`ObjectManager` 批量上传对象变换和标志 |
| `attribute.h/cpp` | 核心 | 属性系统：`Attribute`（单个属性数据）、`AttributeSet`（属性集合）、`AttributeRequest`/`AttributeRequestSet`（着色器属性请求）。支持 Float/Float2/Float3/Float4/UChar4/Transform 类型 |
| `procedural.h/cpp` | 核心 | `Procedural` 抽象基类（同时继承 `Node` 和 `NodeOwner`），在渲染前通过 `generate()` 动态创建节点；`ProceduralManager` 触发更新 |
| `particles.h/cpp` | 核心 | `ParticleSystem`（粒子系统）和 `Particle` 结构体，存储粒子的年龄、生命周期、位置、旋转、速度等；`ParticleSystemManager` 上传粒子数据 |

### 着色（Shading）

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `shader.h/cpp` | 核心 | `Shader` 类：持有 `ShaderGraph`，描述表面、体积和位移三个输出端口；管理发射估算、闭包特征标志（has_surface/has_volume/has_displacement 等）；`ShaderManager` 虚基类定义 SVM/OSL 两种后端的公共接口 |
| `shader_graph.h/cpp` | 核心 | `ShaderGraph`（继承 `NodeOwner`）：着色器节点图的容器，管理 `ShaderNode`、`ShaderInput`、`ShaderOutput` 的连接、断开与重链接；提供图简化、常量折叠、去重、bump-from-displacement 等优化流程 |
| `shader_nodes.h/cpp` | 核心 | 所有内置着色器节点的声明与实现（数百个节点类型），如 `ImageTextureNode`、`NoiseTextureNode`、`PrincipledBsdfNode`、`EmissionNode` 等 |
| `shader.tables` | 数据 | 着色器预计算查找表数据 |
| `svm.h/cpp` | 核心 | `SVMShaderManager`（继承 `ShaderManager`）：将着色器图编译为 SVM（Shader Virtual Machine）字节码；`SVMCompiler` 实现栈分配、节点代码生成、闭包编译 |
| `osl.h/cpp` | 核心 | `OSLManager`：管理 OSL 着色系统的初始化、纹理系统、着色器编译与加载；`OSLShaderManager`（继承 `ShaderManager`）：通过 OSL ShadingSystem 编译着色器；`OSLCompiler`：将着色器图翻译为 OSL 着色器组 |
| `constant_fold.h/cpp` | 优化 | 常量折叠优化：在编译前将可静态求值的着色器节点折叠为常量值 |

### 光照（Lighting）

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `light.h/cpp` | 核心 | `Light` 类（继承 `Geometry`）：支持点光源、日光、面光源、聚光灯等类型（`LightType`），含强度、尺寸、角度、投影、MIS 等参数；`LightManager` 管理光源启用检测、分布更新、光源树构建、背景光重要性采样映射、IES 纹理 |
| `light_tree.h/cpp` | 核心 | `LightTree`：基于 Surface Area Orientation Heuristic (SAOH) 的光源 BVH 树，用于高效光源重要性采样；核心数据结构包括 `OrientationBounds`（方向锥）、`LightTreeMeasure`（包围盒+方向锥+能量）、`LightTreeEmitter`（发射体）、`LightTreeNode`（叶/内/实例节点，使用 `std::variant`）、`LightTreeBucket`（分裂代价桶）；支持光链接（Light Linking）位掩码 |
| `light_tree_debug.h/cpp` | 调试 | 光源树的调试可视化工具 |
| `background.h/cpp` | 核心 | `Background` 类：环境背景着色器，控制是否使用着色器、透明度、粗糙度阈值、光组（lightgroup） |

### 相机与胶片（Camera & Film）

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `camera.h/cpp` | 核心 | `Camera` 类：支持透视/正交/全景（等距柱形、鱼眼、中心柱面等多种投影）相机；参数包括焦距、光圈（景深）、快门（运动模糊）、卷帘快门、立体视觉；计算屏幕-世界/光栅-世界等变换矩阵；支持 OSL 自定义相机脚本 |
| `film.h/cpp` | 核心 | `Film` 类：定义成像平面参数——曝光、滤波器（Box/Gaussian/Blackman-Harris）、雾通道、Cryptomatte、阴影捕获器近似；管理渲染通道（Pass）的自动添加与最终化 |
| `pass.h/cpp` | 核心 | `Pass` 类：渲染通道定义（颜色、法线、深度、运动向量、降噪等），含 `PassInfo` 描述各通道的分量数、滤波、曝光、组合规则；提供静态查找/偏移计算工具函数 |

### 图像与纹理（Image & Texture）

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `image.h/cpp` | 核心 | 图像管理核心：`ImageLoader`（抽象加载器基类）、`ImageHandle`（引用计数句柄）、`ImageManager`（全局图像管理器，负责加载、去重、设备上传、动画帧更新）、`ImageParams`/`ImageMetaData`（参数与元数据） |
| `image_oiio.h/cpp` | 实现 | 基于 OpenImageIO 的图像加载器实现，支持各种文件格式 |
| `image_sky.h/cpp` | 实现 | 天空模型程序化图像加载器（Nishita / Hosek-Wilkie 天空模型） |
| `image_vdb.h/cpp` | 实现 | OpenVDB 体积数据加载器，支持 NanoVDB 设备格式转换 |
| `colorspace.h/cpp` | 核心 | `ColorSpaceManager`：色彩空间检测与转换，支持 sRGB/线性/Raw 等色彩空间，提供像素批量转换和 OSL 纹理缓存处理器 |

### 积分器（Integrator）

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `integrator.h/cpp` | 核心 | `Integrator` 类：路径追踪核心参数——最大弹射次数（漫射/光泽/透射/体积分通道）、体积步进、路径引导（Guiding）、焦散控制、采样模式（Sobol/PMJ）、自适应采样、降噪器配置（OIDN/OptiX） |
| `tabulated_sobol.h/cpp` | 数据 | 预计算的 Sobol 序列查找表，用于低差异采样 |

### 管理基础设施（Infrastructure）

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `scene.h/cpp` | 核心 | `Scene` 类（继承 `NodeOwner`）：场景根对象，持有所有数据列表（cameras/shaders/geometry/objects/lights 等）和所有 Manager 实例；提供统一的 `create_node<T>()`/`delete_node()` 模板接口、增量更新流水线 `update()`、设备数据同步 `device_update()` |
| `devicescene.h/cpp` | 核心 | `DeviceScene`：内核端扁平数据容器，持有全部 `device_vector`（BVH 节点、三角网格、曲线、点云、对象、光源、光源树、粒子、SVM 节点、查找表、体积八叉树等），由各 Manager 写入 |
| `stats.h/cpp` | 工具 | 渲染统计系统：`RenderStats`（网格内存、图像内存、内核采样概况）、`SceneUpdateStats`（各子系统更新耗时）、辅助类 `NamedSizeStats`/`NamedTimeStats`/`NamedNestedSampleStats` |
| `tables.h/cpp` | 工具 | `LookupTables`：统一管理着色器预计算查找表（如 BSDF 预积分表、薄膜干涉表），以固定 256 大小的块分配到设备查找表数组中 |
| `bake.h/cpp` | 工具 | `BakeManager`：纹理烘焙管理器，控制烘焙模式、相机使用、种子设置 |
| `CMakeLists.txt` | 构建 | CMake 构建配置 |

## 核心类与数据结构

### Scene（场景根对象）
```
Scene : NodeOwner
  +-- camera, film, background, integrator  （活动实例指针）
  +-- cameras[], shaders[], geometry[], objects[], ...  （unique_ptr_vector 数据列表）
  +-- image_manager, light_manager, shader_manager, ...  （Manager 实例）
  +-- dscene: DeviceScene  （设备端数据容器）
  +-- bvh: BVH  （顶层加速结构）
```

### Geometry 继承体系
```
Node
  +-- Geometry  （抽象基类：属性集、BVH、运动模糊）
        +-- Mesh  （三角网格 + 细分曲面）
        |     +-- Volume  （体积，扩展 Mesh 增加步长/八叉树）
        +-- Hair  （曲线/毛发）
        +-- PointCloud  （点云）
        +-- Light  （光源几何）
```

### Shader 编译流水线
```
ShaderGraph (节点图)
  --> constant_fold  (常量折叠)
  --> simplify  (图简化/去重)
  --> finalize  (bump 转换、闭包树重写)
  --> SVMCompiler / OSLCompiler  (编译为字节码或 OSL 着色器组)
  --> DeviceScene::svm_nodes / OSL ShadingSystem  (设备端数据)
```

### LightTree（光源树）
```
LightTree
  +-- local_lights_[]    （局部光源发射体）
  +-- distant_lights_[]  （远距光源发射体）
  +-- mesh_lights_[]     （网格发射体）
  +-- root_: LightTreeNode
        +-- Inner: children[2]  （内节点，两个子节点）
        +-- Leaf: [first_emitter, num_emitters]  （叶节点）
        +-- Instance: reference  （实例节点，共享子树）
  每个节点存储 LightTreeMeasure (BoundBox + OrientationBounds + energy)
  分裂策略: SAOH (Surface Area Orientation Heuristic)
```

## 模块架构

场景更新的整体流程如下：

```
宿主应用 (Blender / Hydra)
    |
    v
Scene::create_node / delete_node / 修改属性
    |
    v
各 Manager::tag_update (设置 dirty flags)
    |
    v
Scene::update (Progress)
    |
    +-- ProceduralManager::update  (生成程序化几何)
    +-- ShaderManager::device_update_pre  (预编译着色器)
    +-- GeometryManager::device_update  (几何 + BVH + 属性)
    +-- ObjectManager::device_update  (对象变换 + 标志)
    +-- LightManager::device_update  (光源 + 光源树 + 背景)
    +-- ParticleSystemManager::device_update
    +-- Film::device_update / Camera::device_update
    +-- Integrator::device_update
    +-- ShaderManager::device_update_post  (后处理)
    |
    v
DeviceScene (扁平化设备向量 --> GPU/CPU 内核)
```

## 依赖关系

### 上游依赖（本模块依赖）

| 模块 | 说明 |
|------|------|
| `graph/` | `Node`、`NodeType`、`SocketType` 基础设施，所有场景元素的基类 |
| `bvh/` | BVH 构建算法（BVH2、Embree、OptiX）和参数 |
| `subd/` | 细分曲面算法（`DiagSplit`、`EdgeDice`） |
| `kernel/types.h` | 内核数据类型定义（`KernelObject`、`KernelLight`、`KernelShader` 等） |
| `device/` | 设备抽象层、设备内存管理（`device_vector`、`device_texture`）、降噪参数 |
| `integrator/` | 自适应采样（`AdaptiveSampling`）、路径引导（`GuidingParams`） |
| `util/` | 基础工具库（向量、变换、线程、字符串等） |

### 下游依赖（依赖本模块）

| 模块 | 说明 |
|------|------|
| `session/` | 渲染会话管理器，通过 `Scene` 控制渲染流程 |
| `hydra/` | USD/Hydra 渲染委托，通过 `Scene` API 同步 Hydra 场景数据 |
| `blender/` | Blender 集成层，将 Blender 数据转换为 Cycles 场景元素 |
| `kernel/` | 渲染内核，消费 `DeviceScene` 中的设备向量执行路径追踪 |

## 关键算法与实现细节

### 增量更新与脏标志
每个 Manager 维护 `uint32_t update_flags` 位掩码（如 `MESH_ADDED`、`SHADER_MODIFIED`、`TRANSFORM_MODIFIED`），场景修改时由 `tag_update()` 设置。`device_update()` 仅处理标记的变更，避免全量重新上传。`GeometryManager` 额外使用细粒度标志区分数据修改（`DEVICE_MESH_DATA_MODIFIED`）和需要重新分配（`MESH_DATA_NEED_REALLOC`）。

### 光源树构建（SAOH）
`LightTree` 将光源（含发光三角面、内置光源、发光网格）组织为 BVH 风格的二叉树。分裂代价函数使用 **Surface Area Orientation Heuristic**：`cost = energy * area_measure * orientation_measure`。方向锥（`OrientationBounds`）由轴向量 + theta_o（法线角度范围）+ theta_e（发射角度范围）组成。分裂采用 12 个桶的扫描算法，在 7 个维度（3 空间 + 3 方向 + 1 能量）上选择最优分裂。支持多线程并行构建（`MIN_EMITTERS_PER_THREAD = 4096`）。

### SVM 编译
`SVMCompiler` 将着色器图编译为线性 `int4` 指令数组。编译流程：
1. 按闭包分支遍历节点图，识别共享子树；
2. 为每个节点分配栈偏移（`Stack`，固定 `SVM_STACK_SIZE`）；
3. 逐节点调用 `ShaderNode::compile(SVMCompiler&)` 生成指令；
4. 支持 bump 中心/dx/dy 三次评估的栈状态管理。

### 体积管理
`VolumeManager` 为每个 (Object, Shader) 对构建八叉树（Octree），用于加速异构体积的空跳（null scattering / empty space skipping）。支持基于 OpenVDB 的 SDF 网格生成，以确定网格体积的内部区域。

### 图像管理与去重
`ImageManager` 通过 `ImageLoader::equals()` 比较避免重复加载相同图像。图像采用插槽（slot）系统管理引用计数，支持 OIIO 文件加载、OpenVDB 体积、程序化天空模型三种加载器后端。

## 参见

- `src/kernel/` - 路径追踪渲染内核，消费本模块输出的设备数据
- `src/bvh/` - 层次包围体 (BVH) 构建与遍历算法
- `src/subd/` - 细分曲面算法
- `src/graph/` - 节点系统基础设施（Node/NodeType/SocketType）
- `src/session/` - 渲染会话管理
- `src/hydra/` - USD/Hydra 渲染委托，Cycles 场景的另一入口
- `src/integrator/` - 自适应采样与路径引导
