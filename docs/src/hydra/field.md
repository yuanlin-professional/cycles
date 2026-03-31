# field.h / field.cpp - Hydra渲染代理的体积场缓冲区图元(Bprim)实现

## 概述

本文件实现了 Cycles 渲染器在 Hydra渲染代理框架中的体积场数据适配层。`HdCyclesField` 作为缓冲区图元(Bprim)，负责从通用场景描述(USD)的体积场引用中加载 OpenVDB 文件，并将其注册到 Cycles 的图像管理器中。该功能依赖 `WITH_OPENVDB` 编译选项，未启用时同步操作为空。

## 类与结构体

### HdCyclesField

- **继承**: `PXR_NS::HdField`
- **功能**: 管理 Hydra 体积场到 Cycles 图像句柄的映射，负责 OpenVDB 网格的加载和生命周期管理。
- **关键成员**:
  - `_handle` (`CCL_NS::ImageHandle`) — Cycles 图像管理器返回的图像句柄，供体积着色器引用
- **关键方法**:
  - `Sync()` — 从场景委托获取文件路径和字段名，加载 OpenVDB 网格并注册到图像管理器
  - `GetImageHandle()` — 返回已加载的图像句柄，供体积渲染使用
  - `GetInitialDirtyBitsMask()` — 返回 `DirtyParams`

### HdCyclesVolumeLoader (内部类，仅在 WITH_OPENVDB 下)

- **继承**: `VDBImageLoader`
- **功能**: 自定义的 OpenVDB 加载器，直接从文件路径读取指定网格。
- **关键实现**:
  - 构造函数中直接打开 VDB 文件并读取指定名称的网格
  - 禁用延迟加载和文件拷贝（`delay_load = false`, `setCopyMaxBytes(0)`），避免网络驱动器上的性能问题
  - 捕获 `openvdb::IoError` 和通用异常，输出错误日志

## 核心函数

### `HdCyclesField::Sync()`
- **功能**: 体积场同步的核心逻辑：
  1. 从场景委托获取 `filePath`（`SdfAssetPath` 类型）
  2. 优先使用已解析路径，回退到原始资源路径
  3. 获取 `fieldName`（OpenVDB 网格名称）
  4. 创建 `HdCyclesVolumeLoader` 加载器
  5. 在场景锁保护下，通过 `image_manager->add_image()` 注册图像
- **PXR 版本兼容**: PXR < 2108 使用私有 token `_tokens->fieldName`，PXR >= 2108 使用 `HdFieldTokens->fieldName`

## 依赖关系

- **内部头文件**:
  - `hydra/config.h` — 命名空间配置
  - `scene/image.h` — `ImageHandle`, `ImageParams`, `ImageLoader`
  - `hydra/session.h` — `SceneLock`（仅在 WITH_OPENVDB 下）
  - `scene/image_vdb.h` — `VDBImageLoader` 基类（仅在 WITH_OPENVDB 下）
  - `scene/scene.h` — Cycles 场景（仅在 WITH_OPENVDB 下）
  - `util/log.h` — 日志输出
- **外部头文件**:
  - `pxr/imaging/hd/field.h` — Hydra 场基类
  - `pxr/imaging/hd/sceneDelegate.h` — 场景委托
  - `pxr/usd/sdf/assetPath.h` — USD 资源路径
- **被引用**:
  - `hydra/field.cpp`, `hydra/volume.cpp`, `hydra/render_delegate.cpp`

## 实现细节 / 关键算法

1. **条件编译**: 整个 VDB 加载逻辑被 `#ifdef WITH_OPENVDB` 包裹。未启用时，`Sync()` 仅清除脏位，不执行任何操作。

2. **网络驱动器优化**: `HdCyclesVolumeLoader` 禁用了 OpenVDB 的延迟加载和文件拷贝机制（`setCopyMaxBytes(0)`），因为这些特性在网络文件系统上性能极差。

3. **图像参数**: 注册图像时使用 `frame = 0.0f` 和 `false`（非平铺模式），适合体积数据的一次性加载场景。

## 关联文件

- `src/hydra/volume.cpp` — 体积图元使用 `GetImageHandle()` 获取体积数据
- `src/hydra/render_delegate.cpp` — 注册场类型到渲染代理
- `src/scene/image_vdb.h` — `VDBImageLoader` 基类定义
- `src/scene/image.h` — Cycles 图像管理系统
