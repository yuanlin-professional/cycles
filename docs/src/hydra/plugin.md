# plugin.h / plugin.cpp - Hydra渲染器插件注册入口

## 概述

本文件实现了 Cycles 渲染器在 Pixar Hydra 框架中的插件注册入口。`HdCyclesPlugin` 是 Hydra 渲染器插件的标准接口实现，负责初始化 Cycles 运行时路径、配置日志系统，并提供渲染委托（Render Delegate）的工厂方法。

该文件是 Cycles 作为通用场景描述（USD） Hydra 渲染后端被发现和加载的关键入口点。

## 类与结构体

### HdCyclesPlugin

- **继承**: `PXR_NS::HdRendererPlugin`
- **功能**: Hydra 渲染器插件的标准接口，负责插件初始化和渲染委托的创建/销毁
- **关键方法**:
  - 构造函数 -- 初始化 Cycles 资源路径（相对于插件资源目录），根据环境变量配置日志
  - `IsSupported()` -- 始终返回 `true`，表示 Cycles 插件可用
  - `CreateRenderDelegate()` -- 创建 `HdCyclesDelegate` 实例（两个重载：无参和带设置映射）
  - `DeleteRenderDelegate()` -- 销毁渲染委托实例

## 核心函数

- `TF_REGISTRY_FUNCTION(TfType)` -- 在 USD 类型系统中注册 `HdCyclesPlugin`，使 Hydra 框架能够自动发现 Cycles 渲染后端

## 依赖关系

- **内部头文件**: `hydra/plugin.h`, `hydra/render_delegate.h`, `util/log.h`, `util/path.h`
- **外部依赖**: `pxr/imaging/hd/rendererPlugin.h`, `pxr/imaging/hd/rendererPluginRegistry.h`, `pxr/base/plug/plugin.h`, `pxr/base/plug/thisPlugin.h`, `pxr/base/tf/envSetting.h`, `pxr/base/arch/fileSystem.h`
- **被引用**: 无（作为动态库入口，由 Hydra 框架通过 `plugInfo.json` 加载）

## 实现细节 / 关键算法

1. **路径初始化**: 构造函数通过 `PLUG_THIS_PLUGIN` 获取当前插件的资源路径，转换为绝对路径后调用 `ccl::path_init()` 初始化 Cycles 的资源查找根目录。这确保了内核文件、OSL 着色器等资源能被正确找到。

2. **日志配置**: 通过两个环境变量控制：
   - `CYCLES_LOGGING` (bool) -- 是否启用 Cycles 日志
   - `CYCLES_LOGGING_LEVEL` (string, 默认 "warning") -- 日志级别

3. **命名空间处理**: 为避免 USD 类型系统中命名空间冒号导致的工具兼容性问题，`HdCyclesPlugin` 被放置在 `pxr` 命名空间中（而非 `HdCycles` 命名空间），使得 `plugInfo.json` 中的类型名称不含 `::` 分隔符。

4. **版本兼容**: `IsSupported()` 方法在 PXR_VERSION >= 2302 时增加了 `gpuEnabled` 参数（当前被忽略，始终返回 true）。

## 关联文件

- `hydra/render_delegate.h` -- `HdCyclesDelegate` 渲染委托，由本插件创建
- `hydra/config.h` -- 命名空间和前向声明
- `plugInfo.json` -- USD 插件元数据描述文件，声明 Cycles 渲染后端
- `CMakeLists.txt` -- 将本文件编译为 `hdCycles` 动态库
