# material.h / material.cpp - Hydra渲染代理的材质场景图元(Sprim)实现

## 概述

本文件实现了 Cycles 渲染器在 Hydra渲染代理框架中的材质适配层。`HdCyclesMaterial` 作为场景图元(Sprim)，负责将通用场景描述(USD)的材质网络（包括 UsdPreviewSurface 和 Cycles 原生节点）转换为 Cycles 的着色器图（ShaderGraph）。该文件同时包含 `UsdToCyclesMapping` 映射机制，实现了 USD 着色器节点到 Cycles 节点类型及参数名的自动映射。

## 类与结构体

### HdCyclesMaterial

- **继承**: `PXR_NS::HdMaterial`
- **功能**: 管理 Hydra 材质到 Cycles 着色器的完整转换，支持增量参数更新和完整着色器图重建。
- **关键成员**:
  - `_shader` (`CCL_NS::Shader*`) — Cycles 着色器对象
  - `_nodes` (`unordered_map<SdfPath, NodeDesc>`) — 已创建的着色器节点映射表，按 USD 路径索引
- **关键方法**:
  - `Sync()` — 同步材质资源，根据脏位决定进行参数更新还是完整重建
  - `PopulateShaderGraph()` — 从 `HdMaterialNetwork2` 构建完整的 Cycles 着色器图
  - `UpdateParameters()` — 增量更新已有节点的参数值（三个重载版本）
  - `UpdateConnections()` — 建立着色器节点之间的连接关系
  - `GetCyclesShader()` — 返回底层 Cycles 着色器指针

### NodeDesc (内部结构体)

- **功能**: 描述一个已创建的着色器节点及其映射关系
- **关键成员**:
  - `node` (`ShaderNode*`) — Cycles 着色器节点指针
  - `mapping` (`UsdToCyclesMapping*`) — 对应的 USD 到 Cycles 参数映射

### UsdToCyclesMapping

- **功能**: 将 UsdPreviewSurface 系列节点的类型和参数名映射到 Cycles 等效节点
- **关键方法**:
  - `nodeType()` — 返回对应的 Cycles 节点类型名
  - `parameterName()` — 将 USD 参数/输出名转换为 Cycles 输入名，支持基于连接类型的智能推断

### UsdToCyclesTexture

- **继承**: `UsdToCyclesMapping`
- **功能**: 扩展映射以处理纹理特有的参数转换（如 `wrapS`/`wrapT` -> `extension`，`repeat` -> `periodic`）

## 核心函数

### `PopulateShaderGraph()`
- **功能**: 材质网络到着色器图的核心转换：
  1. 遍历所有节点，根据类型创建 Cycles 节点（支持 `cycles:*` 前缀的原生节点和 USD 标准节点）
  2. 设置每个节点的参数值
  3. 建立节点间的连接
  4. 将终端节点连接到图输出（Surface、Volume、Displacement）
  5. 自动创建 `instanceId` AOV 输出节点

### `UpdateConnections()`
- **功能**: 处理单个节点的所有输入连接。查找上游节点的输出端口和当前节点的输入端口，通过名称映射建立连接。

## 依赖关系

- **内部头文件**:
  - `hydra/config.h` — 命名空间配置
  - `hydra/node_util.h` — `SetNodeValue()` 用于设置节点参数
  - `hydra/session.h` — `SceneLock` 和 `HdCyclesSession`
  - `scene/scene.h`, `scene/shader.h`, `scene/shader_graph.h`, `scene/shader_nodes.h` — Cycles 着色器系统
- **外部头文件**:
  - `pxr/imaging/hd/material.h`, `pxr/imaging/hd/sceneDelegate.h` — Hydra 材质基类
- **被引用**:
  - `hydra/material.cpp`, `hydra/render_delegate.cpp`, `hydra/geometry.inl`

## 实现细节 / 关键算法

1. **节点类型解析**: 节点类型 ID 前缀为 `cycles_` 或 `cycles:` 时，去除前缀直接查找 Cycles 原生节点类型；否则通过 `UsdToCycles` 映射表查找 USD 节点到 Cycles 节点的对应关系。

2. **增量更新优化**: 当只有参数脏（`DirtyParams`）而资源未脏（`DirtyResource`）且节点映射表非空时，仅更新参数值而不重建整个着色器图，显著提升性能。

3. **参数名智能映射**: `UsdToCyclesMapping::parameterName()` 根据连接目标的 socket 类型推断输出名。例如 `result` 输出在连接到标量输入时映射为 `alpha`，连接到向量输入时映射为 `color`。

4. **USD 标准材质映射**:
   - `UsdPreviewSurface` -> `principled_bsdf`（参数映射如 `diffuseColor` -> `base_color`）
   - `UsdUVTexture` -> `image_texture`（参数映射如 `st` -> `vector`，`file` -> `filename`）
   - `UsdPrimvarReader_*` -> `attribute`（参数映射如 `varname` -> `attribute`）

5. **终端连接**: 支持 USD 标准终端（`surface`、`displacement`、`volume`）和 Cycles 自定义终端（`cycles:surface`、`cycles:displacement`、`cycles:volume`），根据节点类型自动推断默认输出端口名。

## 关联文件

- `src/hydra/node_util.h` — 节点参数设置/获取工具
- `src/hydra/geometry.inl` — 几何图元中获取材质着色器
- `src/hydra/render_delegate.cpp` — 注册材质类型到渲染代理
- `src/scene/shader_nodes.h` — 所有可用的 Cycles 着色器节点定义
