# light.h / light.cpp - Hydra渲染代理的光源场景图元(Sprim)实现

## 概述

本文件实现了 Cycles 渲染器在 Hydra渲染代理框架中的光源适配层。`HdCyclesLight` 作为场景图元(Sprim)，负责将通用场景描述(USD)中的各种光源类型（环境光、平行光、圆盘光、矩形光、球形光/聚光灯）转换为 Cycles 内部的 `Light` 和 `Object` 节点。该类还构建对应的着色器图，支持纹理贴图、IES 光域文件和色温等高级光照特性。

## 类与结构体

### HdCyclesLight

- **继承**: `PXR_NS::HdLight`
- **功能**: 管理 Hydra 光源到 Cycles 光源的完整生命周期，包括创建、同步参数、构建着色器图和清理资源。
- **关键成员**:
  - `_object` (`CCL_NS::Object*`) — Cycles 场景中的对象节点，承载光源几何体
  - `_light` (`CCL_NS::Light*`) — Cycles 内部光源节点
  - `_lightType` (`TfToken`) — 光源类型标识（domeLight、distantLight、diskLight、rectLight、sphereLight）
- **关键方法**:
  - `Sync()` — 从场景委托同步变换和光源参数（颜色、曝光、强度、可见性、阴影、形状参数等）
  - `Initialize()` — 延迟初始化：创建 `Object`、`Light` 和默认 `Shader`，根据光源类型设置 Cycles 光源模式
  - `PopulateShaderGraph()` — 构建光源着色器图，处理环境贴图、IES 文件、纹理贴图和色温
  - `Finalize()` — 从场景中删除 `Light` 和 `Object` 节点

## 核心函数

### `Initialize()`
- **功能**: 根据 `_lightType` 创建对应的 Cycles 光源类型：
  - `domeLight` -> `LIGHT_BACKGROUND`
  - `distantLight` -> `LIGHT_DISTANT`
  - `diskLight` -> `LIGHT_AREA`（椭圆形）
  - `rectLight` -> `LIGHT_AREA`（矩形）
  - `sphereLight` -> `LIGHT_POINT`
- 默认启用 MIS（多重重要性采样），默认对相机不可见

### `PopulateShaderGraph()`
- **功能**: 根据光源类型和参数动态构建着色器图：
  - 环境光使用 `BackgroundNode`，将强度烘焙到节点中
  - 其他光源使用 `EmissionNode` 或自定义衰减节点（`LightFalloffNode`）
  - 支持 `BlackbodyNode`（色温）、`EnvironmentTextureNode`/`ImageTextureNode`（纹理贴图）、`IESLightNode`（光域文件）
  - 多种效果可组合叠加（如色温 + 纹理）

## 依赖关系

- **内部头文件**:
  - `hydra/config.h` — 命名空间配置
  - `hydra/session.h` — 获取舞台单位和场景锁
  - `kernel/types.h` — `PATH_RAY_*` 可见性标志
  - `scene/light.h`, `scene/object.h`, `scene/scene.h` — Cycles 场景节点
  - `scene/shader.h`, `scene/shader_graph.h`, `scene/shader_nodes.h` — 着色器图构建
  - `util/hash.h`, `util/transform.h` — 哈希和变换工具
- **外部头文件**:
  - `pxr/imaging/hd/light.h`, `pxr/imaging/hd/sceneDelegate.h` — Hydra 光源基类
  - `pxr/usd/sdf/assetPath.h` — USD 资源路径
- **被引用**:
  - `hydra/light.cpp`, `hydra/render_delegate.cpp`

## 实现细节 / 关键算法

1. **强度计算**: 最终强度 = `color * 2^exposure * intensity`。非平行光额外乘以 `M_PI` 将辐照度转换为辐射通量；平行光乘以 4.0 以近似匹配 Karma 渲染器的效果。

2. **聚光灯检测**: 球形光源带有 `shapingConeAngle` 或 `shapingConeSoftness` 参数时，自动切换为 `LIGHT_SPOT` 类型。

3. **着色器图动态更新**: 当变换改变且光源为环境光或着色器包含空间变化节点时，即使参数未脏也会重新构建着色器图，因为变换可能被烘焙到着色器中（如 `TextureCoordinateNode` 的 `ob_tfm`）。

4. **不可见光源处理**: 不可见光源通过将强度置零来禁用，确保 `LightManager::test_enabled_lights` 正确更新启用标志。

5. **环境贴图方向**: 环境光的纹理坐标节点使用去除平移分量的变换矩阵，只保留旋转信息。

## 关联文件

- `src/hydra/session.h` — `SceneLock` 和 `HdCyclesSession`
- `src/hydra/render_delegate.cpp` — 注册光源类型到渲染代理
- `src/scene/light.h` — Cycles 内部 `Light` 定义
- `src/scene/shader_nodes.h` — 各种着色器节点类型
