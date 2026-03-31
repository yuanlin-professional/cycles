# render_delegate.h / render_delegate.cpp - Cycles Hydra渲染代理核心委托实现

## 概述

本文件实现了 Cycles 渲染器在 Pixar Hydra 渲染框架中的渲染委托（Render Delegate），是整个 Hydra渲染代理 集成的核心入口。`HdCyclesDelegate` 负责管理所有渲染图元（Rprim）、场景图元（Sprim）、缓冲区图元（Bprim）的创建与销毁，并提供渲染设置的读写接口。

该类是 Hydra 框架与 Cycles 渲染引擎之间的桥梁，将通用场景描述（USD）场景数据映射到 Cycles 的内部表示。

## 类与结构体

### HdCyclesDelegate

- **继承**: `PXR_NS::HdRenderDelegate`
- **功能**: 作为 Hydra 渲染委托，负责创建和管理渲染通道、实例化器、各类图元（Rprim/Sprim/Bprim），以及渲染设置的存取。
- **关键成员**:
  - `_hgi` (`Hgi*`) -- Hydra 图形接口指针，用于判断是否支持 OpenGL 显示驱动
  - `_renderParam` (`unique_ptr<HdCyclesSession>`) -- Cycles 会话封装，持有 `Session` 和 `Scene` 的生命周期
- **关键方法**:
  - `CreateRprim()` -- 根据类型标识创建渲染图元（Rprim），支持 mesh、basisCurves、points、volume
  - `CreateSprim()` -- 创建场景图元（Sprim），支持 camera、material、各类灯光、extComputation
  - `CreateBprim()` -- 创建缓冲区图元（renderBuffer、openvdbAsset）
  - `CreateRenderPass()` -- 创建 `HdCyclesRenderPass` 渲染通道实例
  - `CreateInstancer()` -- 创建 `HdCyclesInstancer` 实例化器
  - `CommitResources()` -- 在场景锁保护下调用 `UpdateScene()` 提交资源变更
  - `SetRenderSetting()` / `GetRenderSetting()` -- 读写渲染参数（采样数、时间限制、设备、积分器属性等）
  - `GetRenderStats()` -- 返回渲染统计信息（进度、内存、耗时）
  - `GetDefaultAovDescriptor()` -- 返回 AOV（color/depth/normal/primId 等）的默认格式描述
  - `IsDisplaySupported()` -- 判断是否支持实时显示驱动（仅 Windows + OpenGL）
  - `Pause()` / `Resume()` -- 暂停与恢复渲染会话
  - `GetMaterialRenderContexts()` -- 返回材质网络选择器标识 `cycles`

### HdCyclesRenderSettingsTokens

- **功能**: 定义渲染设置 Token 常量，包括 `stageMetersPerUnit`、`cycles:device`、`cycles:threads`、`cycles:time_limit`、`cycles:samples`、`cycles:sample_offset`

## 核心函数

- `GetSessionParams()` -- 匿名命名空间内的辅助函数，从 `HdRenderSettingsMap` 中提取线程数和设备类型，构建 `SessionParams`。支持从环境变量 `CYCLES_DEVICE` 读取设备类型，设备类型自动转大写后查找可用设备列表。

## 依赖关系

- **内部头文件**:
  - `hydra/config.h`
  - `hydra/camera.h`, `hydra/curves.h`, `hydra/field.h`, `hydra/instancer.h`, `hydra/light.h`, `hydra/material.h`, `hydra/mesh.h`, `hydra/node_util.h`, `hydra/pointcloud.h`, `hydra/render_buffer.h`, `hydra/render_pass.h`, `hydra/session.h`, `hydra/volume.h`
  - `scene/integrator.h`, `scene/scene.h`, `session/session.h`
- **外部依赖**: `pxr/imaging/hd/renderDelegate.h`, `pxr/imaging/hgi/hgi.h`, `pxr/base/tf/getenv.h`, `pxr/imaging/hd/extComputation.h`
- **被引用**: `render_pass.cpp`, `plugin.cpp`, `file_reader.cpp`

## 实现细节 / 关键算法

1. **设备选择策略**: 优先从渲染设置中读取 `cycles:device`，其次从环境变量 `CYCLES_DEVICE` 获取，最终回退到 CPU。多 GPU 设备通过 `Device::get_multi_device()` 合并。
2. **图元类型注册**: 支持的 Rprim 类型为 mesh、basisCurves、points（volume 需 `WITH_OPENVDB` 宏）；Sprim 类型包含 camera、material 和五种灯光类型；Bprim 类型为 renderBuffer 和 openvdbAsset。
3. **渲染设置传递**: 构造时跳过 `device` 和 `threads`（仅用于初始化），其余设置通过 `SetRenderSetting()` 逐一应用。积分器设置通过 socket 名称动态映射（前缀 `cycles:integrator:`）。
4. **AOV 格式**: color 在支持显示驱动时使用 `Float16Vec4`，否则使用 `Float32Vec4`；depth 使用 `Float32`；ID 类 AOV 使用 `Int32`。

## 关联文件

- `hydra/session.h` -- `HdCyclesSession` 的定义
- `hydra/render_pass.h` -- 由 `CreateRenderPass()` 创建
- `hydra/plugin.h` -- 插件入口，通过 `CreateRenderDelegate()` 实例化本类
- `hydra/node_util.h` -- `GetNodeValue()` / `SetNodeValue()` 辅助函数
