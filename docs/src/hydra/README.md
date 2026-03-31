# Hydra 模块 - Cycles 的 USD/Hydra 渲染委托

## 概述

`src/hydra/` 实现了 Cycles 渲染引擎的 **Hydra Render Delegate**（渲染委托），使 Cycles 可以作为 Pixar USD（Universal Scene Description）生态系统中的渲染后端。通过此模块，任何支持 Hydra 渲染框架的应用程序（如 Pixar usdview、NVIDIA Omniverse、SideFX Solaris 等）均可使用 Cycles 进行路径追踪渲染。

Hydra 是 USD 的渲染抽象层，定义了 Rprim（渲染图元）、Sprim（状态图元）、Bprim（缓冲图元）三种图元类型的接口。本模块的核心工作是将 Hydra 场景图元翻译为 Cycles 的 `scene/` 模块中的原生对象（`Mesh`、`Hair`、`Light`、`Shader` 等），并驱动 Cycles 的 `Session` 执行渲染。

模块由 NVIDIA 和 Blender Foundation 联合开发，命名空间为 `HdCycles`（宏 `HDCYCLES_NAMESPACE_OPEN_SCOPE`）。

## 目录结构

### 插件注册与委托核心

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `plugin.h/cpp` | 核心 | `HdCyclesPlugin`（继承 `HdRendererPlugin`）：Hydra 渲染器插件入口点，负责创建和销毁 `HdCyclesDelegate` 实例；通过 `plugInfo.json` 注册到 USD 插件系统 |
| `plugInfo.json` | 配置 | USD 插件注册元数据，声明渲染器名称和类型 |
| `render_delegate.h/cpp` | 核心 | `HdCyclesDelegate`（继承 `HdRenderDelegate`）：渲染委托主类，实现 Hydra 的全部工厂接口——创建/销毁 Rprim、Sprim、Bprim，管理渲染设置（采样数、设备、线程、时间限制），提交资源变更（`CommitResources`），声明支持的图元类型和材质网络选择器 |
| `config.h` | 配置 | 命名空间定义宏（`HDCYCLES_NAMESPACE_OPEN_SCOPE` / `CLOSE_SCOPE`）和前向声明；定义 `CCL_NS`（ccl）与 `HD_CYCLES_NS`（HdCycles）命名空间映射 |

### 会话与渲染管线

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `session.h/cpp` | 核心 | `HdCyclesSession`（继承 `HdRenderParam`）：封装 Cycles `Session` 对象，管理场景锁（`SceneLock` RAII 类）、AOV 绑定同步、舞台计量单位（`stageMetersPerUnit`）；可接管外部 Session 或自行创建 |
| `render_pass.h/cpp` | 核心 | `HdCyclesRenderPass`（继承 `HdRenderPass`）：渲染通道执行器，在 `_Execute()` 中驱动 Cycles 会话的启动、AOV 配置和收敛检测 |
| `render_buffer.h/cpp` | 核心 | `HdCyclesRenderBuffer`（继承 `HdRenderBuffer`）：渲染缓冲区管理，提供像素数据的分配、映射（Map/Unmap）、收敛状态跟踪；支持 GPU 资源共享（`GetResource`/`SetResource`） |
| `display_driver.h/cpp` | 核心 | `HdCyclesDisplayDriver`（继承 `CCL_NS::DisplayDriver`）：实时显示驱动，通过 HGI（Hydra Graphics Interface）将渲染结果推送到视口；管理 OpenGL PBO 和 HGI 纹理的创建与同步，支持 GPU 图形互操作（Graphics Interop） |
| `output_driver.h/cpp` | 核心 | `HdCyclesOutputDriver`（继承 `CCL_NS::OutputDriver`）：离线输出驱动，将渲染瓦片写入 `HdCyclesRenderBuffer`；实现 `write_render_tile()` 和 `update_render_tile()` |

### 几何图元适配（Rprim）

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `geometry.h/cpp` | 核心 | `HdCyclesGeometry<Base, CyclesBase>` 模板类（继承 Hydra 的 `HdRprim` 派生类）：所有几何适配器的公共基类，处理 `Sync()`（同步脏位）、初始化 Cycles 几何对象和 Object 实例、管理实例化变换 |
| `geometry.inl` | 实现 | `HdCyclesGeometry` 模板的内联实现 |
| `mesh.h/cpp` | 适配 | `HdCyclesMesh`（特化 `HdCyclesGeometry<HdMesh, Mesh>`）：将 Hydra 网格拓扑（`HdMeshTopology`）转换为 Cycles 三角网格；处理顶点、法线、Primvar 属性同步 |
| `curves.h/cpp` | 适配 | `HdCyclesCurves`（特化 `HdCyclesGeometry<HdBasisCurves, Hair>`）：将 Hydra 基础曲线转换为 Cycles 毛发曲线；同步控制点、宽度和 Primvar |
| `pointcloud.h/cpp` | 适配 | `HdCyclesPoints`（特化 `HdCyclesGeometry<HdPoints, PointCloud>`）：将 Hydra 点转换为 Cycles 点云；同步位置、半径和 Primvar |
| `volume.h/cpp` | 适配 | `HdCyclesVolume`（特化 `HdCyclesGeometry<HdVolume, Volume>`）：将 Hydra 体积转换为 Cycles 体积对象 |
| `instancer.h/cpp` | 适配 | `HdCyclesInstancer`（继承 `HdInstancer`）：实例化器，计算实例变换矩阵（组合平移、旋转、缩放和 instanceTransform）；支持嵌套实例化 |

### 状态图元适配（Sprim）

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `camera.h/cpp` | 适配 | `HdCyclesCamera`（继承 `HdCamera`）：将 Hydra 相机（`GfCamera`）参数映射到 Cycles `Camera`，处理投影矩阵、视口适配策略（`ConformWindowPolicy`）、运动模糊采样 |
| `light.h/cpp` | 适配 | `HdCyclesLight`（继承 `HdLight`）：将 Hydra 光源类型（sphere/disk/rect/distant/dome）映射到 Cycles `Light`，动态构建光源着色器图 |
| `material.h/cpp` | 适配 | `HdCyclesMaterial`（继承 `HdMaterial`）：将 USD 材质网络（`HdMaterialNetwork2`）翻译为 Cycles `ShaderGraph`；使用 `UsdToCyclesMapping` 映射表将 USD Preview Surface / MaterialX 节点转换为 Cycles 着色器节点，处理参数和连接 |

### 缓冲图元适配（Bprim）

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `field.h/cpp` | 适配 | `HdCyclesField`（继承 `HdField`）：将 Hydra 场数据（OpenVDB 资产）同步为 Cycles `ImageHandle`，供体积着色器使用 |

### 工具与辅助

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `attribute.h/cpp` | 工具 | `ApplyPrimvars()` 函数：将 Hydra Primvar 值（`VtValue`）转换并写入 Cycles `AttributeSet` |
| `node_util.h/cpp` | 工具 | `SetNodeValue()` / `GetNodeValue()`：在 USD `VtValue` 与 Cycles `Node` socket 值之间进行双向转换 |
| `file_reader.h/cpp` | 工具 | `HdCyclesFileReader`：从 USD 文件读取场景并注入 Cycles `Session`，主要用于独立测试和命令行渲染 |
| `resources/` | 资源 | 插件资源文件目录 |
| `CMakeLists.txt` | 构建 | CMake 构建配置 |

## 核心类与数据结构

### 类继承体系

```
PXR_NS::HdRendererPlugin
  +-- HdCyclesPlugin              (插件入口)

PXR_NS::HdRenderDelegate
  +-- HdCyclesDelegate            (渲染委托主类)

PXR_NS::HdRenderParam
  +-- HdCyclesSession             (Session 封装 + AOV 管理)

PXR_NS::HdRenderPass
  +-- HdCyclesRenderPass          (渲染通道执行)

PXR_NS::HdRenderBuffer
  +-- HdCyclesRenderBuffer        (渲染缓冲区)

HdCyclesGeometry<Base, CyclesBase>  (几何适配模板)
  +-- HdCyclesMesh      <HdMesh, Mesh>
  +-- HdCyclesCurves    <HdBasisCurves, Hair>
  +-- HdCyclesPoints    <HdPoints, PointCloud>
  +-- HdCyclesVolume    <HdVolume, Volume>

PXR_NS::HdCamera    --> HdCyclesCamera
PXR_NS::HdLight     --> HdCyclesLight
PXR_NS::HdMaterial  --> HdCyclesMaterial
PXR_NS::HdField     --> HdCyclesField
PXR_NS::HdInstancer --> HdCyclesInstancer

CCL_NS::DisplayDriver --> HdCyclesDisplayDriver
CCL_NS::OutputDriver  --> HdCyclesOutputDriver
```

### SceneLock（RAII 场景锁）

`SceneLock` 是一个 RAII 辅助类，在构造时从 `HdRenderParam` 获取 Cycles 场景互斥锁，在析构时自动释放。所有修改 Cycles 场景的 Hydra 同步操作都必须在 `SceneLock` 保护下进行，以避免与渲染线程产生数据竞争。

## 模块架构

### Hydra 同步流程

```
USD 舞台 (UsdStage)
    |
    v
Hydra 变更追踪器 (HdChangeTracker)
    |  检测脏位 (DirtyBits)
    v
HdCyclesDelegate
    |
    +-- CreateRprim/Sprim/Bprim  (按需创建适配器)
    |
    +-- 各适配器 Sync()
    |     |
    |     +-- HdCyclesMesh::Populate  --> ccl::Mesh (顶点/面/法线/UV)
    |     +-- HdCyclesCurves::Populate --> ccl::Hair (曲线点/宽度)
    |     +-- HdCyclesPoints::Populate --> ccl::PointCloud (点/半径)
    |     +-- HdCyclesVolume::Populate --> ccl::Volume (OpenVDB 字段)
    |     +-- HdCyclesLight::Sync     --> ccl::Light + ShaderGraph
    |     +-- HdCyclesMaterial::Sync   --> ccl::Shader + ShaderGraph
    |     +-- HdCyclesCamera::Sync     --> ccl::Camera
    |     +-- HdCyclesInstancer::Sync  --> 实例变换矩阵
    |
    +-- CommitResources
    |     |
    |     v
    |   HdCyclesSession::UpdateScene()
    |     --> Scene 各 Manager 执行 device_update
    |
    v
HdCyclesRenderPass::_Execute()
    |
    +-- 配置 AOV 绑定 (SyncAovBindings)
    +-- Session::start() / Session::reset()
    |
    +-- HdCyclesOutputDriver::write_render_tile()
    |     --> HdCyclesRenderBuffer::WritePixels()
    |
    +-- HdCyclesDisplayDriver::draw()
          --> HGI 纹理更新 --> 视口显示
```

### Rprim / Sprim / Bprim 分类

| 类别 | Hydra 类型 | Cycles 适配器 | Cycles 对象 |
|------|-----------|---------------|-------------|
| Rprim | HdMesh | HdCyclesMesh | Mesh + Object |
| Rprim | HdBasisCurves | HdCyclesCurves | Hair + Object |
| Rprim | HdPoints | HdCyclesPoints | PointCloud + Object |
| Rprim | HdVolume | HdCyclesVolume | Volume + Object |
| Sprim | HdCamera | HdCyclesCamera | Camera |
| Sprim | HdLight | HdCyclesLight | Light + Object + Shader |
| Sprim | HdMaterial | HdCyclesMaterial | Shader + ShaderGraph |
| Bprim | HdRenderBuffer | HdCyclesRenderBuffer | (像素缓冲区) |
| Bprim | HdField | HdCyclesField | ImageHandle |

## 依赖关系

### 上游依赖（本模块依赖）

| 模块 | 说明 |
|------|------|
| `scene/` | Cycles 场景描述层——`Mesh`、`Hair`、`Light`、`Shader`、`Camera`、`Film`、`Object`、`Scene` 等全部场景类型 |
| `session/` | `Session`（渲染会话）、`DisplayDriver`（显示驱动基类）、`OutputDriver`（输出驱动基类） |
| `graph/` | `Node`、`SocketType`——用于 `node_util.h` 中的值转换 |
| `util/` | 线程（`thread_scoped_lock`）、基础类型 |
| **pxr (USD)** | Hydra 框架核心——`HdRenderDelegate`、`HdRprim`、`HdSprim`、`HdBprim`、`HdRenderPass`、`HdRenderBuffer`、`HdInstancer`、`HdMaterial`、`Hgi` 等 |

### 下游依赖（依赖本模块）

| 模块 | 说明 |
|------|------|
| USD 应用程序 | 通过 `plugInfo.json` 插件注册机制，任何 Hydra 兼容的应用程序可动态加载本模块 |

## 关键算法与实现细节

### 几何适配模板
`HdCyclesGeometry<Base, CyclesBase>` 是一个 CRTP 风格的模板基类，统一了所有几何图元的同步逻辑：
1. **Initialize**：通过 `Scene::create_node<CyclesBase>()` 创建 Cycles 几何对象和对应的 `Object` 实例；
2. **Sync**：检查 Hydra 脏位，调用子类的 `Populate()` 虚函数填充几何数据；
3. **实例化**：查询 `HdCyclesInstancer` 获取实例变换矩阵，创建/更新多个 `Object` 实例；
4. **Finalize**：清理——调用 `Scene::delete_node()` 删除关联的 Cycles 对象。

### 材质网络翻译
`HdCyclesMaterial` 将 USD 材质网络（`HdMaterialNetwork2`）转换为 Cycles `ShaderGraph`：
1. 遍历网络中的所有节点，通过 `UsdToCyclesMapping` 映射表查找对应的 Cycles `ShaderNode` 类型；
2. 使用 `SetNodeValue()` 将 USD 参数值（`VtValue`）转换为 Cycles socket 值；
3. 根据网络中的连接关系，调用 `ShaderGraph::connect()` 建立节点间的输入/输出连接；
4. 将最终的 `ShaderGraph` 设置到 `Shader` 对象并触发重编译。

### 显示驱动与 GPU 互操作
`HdCyclesDisplayDriver` 实现实时视口渲染：
- 在 Windows 上创建共享 OpenGL 上下文，通过 PBO（Pixel Buffer Object）实现 GPU 到 GPU 的数据传输；
- 使用 HGI 纹理接口（`HgiTextureHandle`）与 Hydra 渲染管线集成；
- 支持 Graphics Interop：CUDA/HIP 渲染结果直接映射到 OpenGL PBO，避免 GPU-CPU-GPU 往返拷贝；
- 双同步栅栏（`gl_render_sync_` / `gl_upload_sync_`）确保渲染与显示的正确时序。

### AOV 绑定管理
`HdCyclesSession` 维护 AOV（Arbitrary Output Variable）绑定列表：
- `SyncAovBindings()` 对比新旧绑定列表，增量添加/移除 Cycles `Pass`；
- 每个 AOV 绑定关联一个 `HdCyclesRenderBuffer`，渲染完成时由 `HdCyclesOutputDriver` 将瓦片像素写入对应缓冲区；
- 显示 AOV 通过 `HdCyclesDisplayDriver` 实时推送到视口。

## 参见

- `src/scene/` - Cycles 原生场景描述层，本模块的数据翻译目标
- `src/session/` - 渲染会话管理，本模块的渲染驱动引擎
- `src/kernel/` - 路径追踪渲染内核
- [USD 官方文档 - Hydra](https://openusd.org/release/api/hd_page_front.html) - Hydra 渲染框架规范
- [HdRendererPlugin](https://openusd.org/release/api/class_hd_renderer_plugin.html) - 渲染器插件接口
