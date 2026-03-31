# file_reader.h / file_reader.cpp - Hydra渲染代理的 USD 文件读取器

## 概述

本文件提供了通过 Hydra渲染代理管线从通用场景描述(USD)文件读取完整场景的功能。`HdCyclesFileReader` 是一个静态工具类，它打开 USD Stage，创建完整的 Hydra 渲染管线（渲染代理、渲染索引、场景委托），执行一次完整的场景同步，并可选地应用第一个找到的相机设置。这是 Cycles 独立模式下加载 USD 文件的入口点。

## 类与结构体

### HdCyclesFileReader

- **继承**: 无
- **功能**: 静态工具类，提供从 USD 文件路径到 Cycles 场景的一站式加载。
- **关键方法**:
  - `read()` — 静态方法，读取 USD 文件并将内容同步到指定的 Cycles 会话中

### DummyHdTask (内部类)

- **继承**: `HdTask`
- **功能**: 占位任务，唯一目的是向渲染索引提供渲染标签（`geometry` 和 `render`），使得场景图元能被正确枚举和同步。
- **关键方法**: `Sync()`、`Prepare()`、`Execute()` 均为空实现；`GetRenderTags()` 返回预设的标签列表

## 核心函数

### `HdCyclesFileReader::read()`

- **签名**: `static void read(Session *session, const char *filepath, const bool use_camera = true)`
- **参数**:
  - `session` — 目标 Cycles 会话，场景数据将被加载到此会话的场景中
  - `filepath` — USD 文件路径
  - `use_camera` — 是否应用 Stage 中第一个相机的设置（默认 true）
- **执行流程**:
  1. 注册 USD 插件（`PlugRegistry::RegisterPlugins`）
  2. 打开 USD Stage（`UsdStage::Open`）
  3. 创建渲染设置，配置 `stageMetersPerUnit`
  4. 创建 `HdCyclesDelegate` 渲染代理（传入现有 Session，`keep_nodes = true`）
  5. 创建 `HdRenderIndex` 和 `UsdImagingDelegate`
  6. 设置渲染集合和占位任务
  7. 填充场景委托（`scene_delegate->Populate`）
  8. 执行完整同步（`render_index->SyncAll`）
  9. 提交资源（`render_delegate.CommitResources`）
  10. 可选：遍历 Stage 查找第一个 `UsdGeomCamera`，通过 `HdCyclesCamera::ApplyCameraSettings` 应用相机

## 依赖关系

- **内部头文件**:
  - `hydra/config.h` — 命名空间配置
  - `hydra/camera.h` — `HdCyclesCamera`（应用相机设置）
  - `hydra/render_delegate.h` — `HdCyclesDelegate`（创建渲染代理）
  - `session/session.h` — Cycles `Session`
  - `scene/scene.h` — Cycles `Scene`
  - `util/log.h`, `util/path.h`, `util/unique_ptr.h` — 工具函数
- **外部头文件**:
  - `pxr/base/plug/registry.h` — USD 插件注册
  - `pxr/imaging/hd/dirtyList.h`, `pxr/imaging/hd/renderDelegate.h`, `pxr/imaging/hd/renderIndex.h` — Hydra 渲染管线
  - `pxr/imaging/hd/rprimCollection.h`, `pxr/imaging/hd/task.h` — 渲染集合和任务
  - `pxr/usd/usd/stage.h` — USD Stage
  - `pxr/usd/usdGeom/camera.h`, `pxr/usd/usdGeom/metrics.h` — USD 几何相机和度量
  - `pxr/usdImaging/usdImaging/delegate.h` — USD 到 Hydra 的场景委托
- **被引用**:
  - `hydra/file_reader.cpp`（仅自身实现）

## 实现细节 / 关键算法

1. **USD 插件发现**: 通过 `path_get("usd")` 获取 Cycles 安装目录下的 USD 插件路径，注册到 `PlugRegistry` 中，确保所有必需的 USD 文件格式和 Schema 插件可用。

2. **PXR 版本兼容**: PXR < 2111 使用 `HdDirtyList` + `EnqueuePrimsToSync` 路径；PXR >= 2111 使用简化的 `EnqueueCollectionToSync` 路径。

3. **keep_nodes 模式**: 渲染代理创建时 `keep_nodes = true`，这意味着当 Hydra 图元在 `Finalize` 时不会删除 Cycles 场景节点。这是因为文件读取完成后，渲染代理和渲染索引会被销毁，但 Cycles 场景节点需要保留供后续渲染使用。

4. **相机查找策略**: 遍历 Stage 中的所有 Prim，找到第一个 `UsdGeomCamera` 类型的 Prim，通过渲染索引获取对应的 HdSprim，然后调用 `ApplyCameraSettings` 设置到会话的相机上。当前尚未支持从 `UsdRenderSettings` 获取指定相机。

5. **渲染标签机制**: `DummyHdTask` 的存在确保渲染索引知道需要同步哪些类型的图元。没有这个任务，渲染索引可能会跳过某些图元的同步。

## 关联文件

- `src/hydra/camera.h` — 相机设置应用
- `src/hydra/render_delegate.h` — `HdCyclesDelegate` 渲染代理
- `src/app/cycles_standalone.cpp`（可能） — 独立模式下调用 `HdCyclesFileReader::read()`
- `src/session/session.h` — Cycles 会话定义
