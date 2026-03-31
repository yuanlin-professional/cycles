# camera.h / camera.cpp - Hydra渲染代理的相机场景图元(Sprim)实现

## 概述

本文件实现了 Cycles 渲染器在 Hydra渲染代理框架中的相机适配层。`HdCyclesCamera` 作为场景图元(Sprim)，负责将通用场景描述(USD)中的相机参数（投影类型、光圈、焦距、裁剪范围、景深等）同步到 Cycles 内部的 `Camera` 对象。该类同时处理了不同 PXR 版本间的 API 差异，确保了广泛的兼容性。

## 类与结构体

### HdCyclesCamera

- **继承**: `PXR_NS::HdCamera`
- **功能**: 将 Hydra 场景委托中的相机数据同步至 Cycles 渲染器的相机对象，支持透视/正交投影、运动模糊变换采样以及单位缩放。
- **关键成员**:
  - `_data` (`GfCamera`) — 存储从场景委托获取的完整相机参数（光圈、焦距、裁剪范围等）
  - `_transformSamples` (`HdTimeSampleArray<GfMatrix4d, 2>`) — 存储变换的时间采样数据，用于运动模糊
- **关键方法**:
  - `Sync()` — 从 `HdSceneDelegate` 同步相机的变换、投影矩阵、窗口策略、裁剪平面和各项相机参数（光圈、焦距、fStop、焦距距离等）
  - `ApplyCameraSettings()` — 将相机参数应用到 Cycles 的 `Camera` 对象（三个重载版本：实例方法、从 `GfCamera` 静态方法、从视图/投影矩阵静态方法）
  - `Finalize()` — 清理相机资源
  - `GetInitialDirtyBitsMask()` — 返回 `AllDirty`，表示首次同步时所有属性需要更新

## 核心函数

### `convert_camera_transform()`
- **签名**: `Transform convert_camera_transform(const GfMatrix4d &matrix, float metersPerUnit)`
- **功能**: 将通用场景描述(USD)的相机变换矩阵转换为 Cycles 内部格式。翻转 Z 轴以匹配 Cycles 的坐标系约定，并根据舞台的米/单位比例缩放平移分量。

### `ApplyCameraSettings()` (静态，从 GfCamera)
- **功能**: 核心相机参数转换逻辑。根据投影类型设置透视/正交模式，通过 `CameraUtilConformWindow` 适配窗口宽高比，计算视口平面、传感器尺寸、FOV、近远裁剪面和景深参数。

## 依赖关系

- **内部头文件**:
  - `hydra/config.h` — Hydra渲染代理命名空间和基础配置
  - `hydra/session.h` — 获取 `HdCyclesSession` 的舞台单位信息
  - `scene/camera.h` — Cycles 内部 `Camera` 类
- **外部头文件**:
  - `pxr/base/gf/camera.h`, `pxr/base/gf/frustum.h` — USD 相机和视锥体
  - `pxr/imaging/hd/camera.h`, `pxr/imaging/hd/sceneDelegate.h` — Hydra 相机基类和场景委托
  - `pxr/imaging/hd/timeSampleArray.h` — 时间采样数组（运动模糊）
- **被引用**:
  - `hydra/camera.cpp`, `hydra/render_pass.cpp`, `hydra/render_delegate.cpp`, `hydra/file_reader.cpp`

## 实现细节 / 关键算法

1. **PXR 版本兼容**: 代码通过 `#if PXR_VERSION` 条件编译处理多个 API 变化点：
   - PXR < 2102: 使用 `DirtyViewMatrix`/`worldToViewMatrix` 路径
   - PXR < 2111: 手动从投影矩阵反推光圈和裁剪范围
   - PXR >= 2102: 使用 `SetFromViewAndProjectionMatrix` 简化路径

2. **坐标系转换**: `convert_camera_transform()` 翻转 Z 轴（`t.*.z *= -1.0f`），因为 USD 使用右手坐标系而 Cycles 相机朝 -Z 方向。

3. **单位缩放**: 所有涉及长度的参数（传感器尺寸、裁剪距离、焦距、对焦距离等）均乘以 `metersPerUnit` 以适配不同 USD 舞台的单位系统。

4. **视口平面归一化**: 透视投影下，视口平面按高度归一化（`viewplane *= 2.0 / viewplane.GetSize()[1]`），确保 Cycles 内部视口平面的一致性。

5. **运动模糊**: `ApplyCameraSettings()` 实例方法将 `_transformSamples` 中的所有时间采样变换传递给 `cam->set_motion()`，支持相机运动模糊。

## 关联文件

- `src/hydra/session.h` — `HdCyclesSession`（提供 `GetStageMetersPerUnit()`）
- `src/hydra/render_pass.cpp` — 渲染通道中调用相机设置
- `src/hydra/file_reader.cpp` — USD 文件读取时应用相机设置
- `src/scene/camera.h` — Cycles 内部相机定义
