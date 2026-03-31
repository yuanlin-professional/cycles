# render_pass.h / render_pass.cpp - Hydra渲染通道执行与会话控制

## 概述

本文件实现了 Cycles 的 Hydra 渲染通道（Render Pass），负责在每一帧将场景数据提交给 Cycles 渲染引擎执行。`HdCyclesRenderPass` 管理 AOV 绑定同步、相机参数更新、视口尺寸设置，并在场景状态变化时触发渲染会话重置与启动。

渲染通道是 Hydra 框架调用渲染引擎进行实际绘制的核心执行路径。

## 类与结构体

### HdCyclesRenderPass

- **继承**: `PXR_NS::HdRenderPass`
- **功能**: 实现 Hydra 渲染通道协议，在 `_Execute` 中驱动 Cycles 会话的帧渲染流程
- **关键成员**:
  - `_renderParam` (`HdCyclesSession*`) -- 指向 Cycles 会话参数对象
  - `_lastSettingsVersion` (`unsigned int`) -- 上次应用的渲染设置版本号，用于检测变更
- **关键方法**:
  - `IsConverged()` -- 检查所有 AOV 渲染缓冲区是否已收敛
  - `_Execute()` -- 核心执行方法，完成 AOV 同步、相机更新、会话重置与启动
  - `ResetConverged()` -- 将所有渲染缓冲区标记为未收敛
  - `_MarkCollectionDirty()` -- 空实现（Cycles 不需要响应集合脏标记）

## 核心函数

无独立核心函数，所有逻辑封装在 `HdCyclesRenderPass` 类方法中。

## 依赖关系

- **内部头文件**:
  - `hydra/render_pass.h`, `hydra/camera.h`, `hydra/output_driver.h`, `hydra/render_buffer.h`, `hydra/render_delegate.h`, `hydra/session.h`
  - `scene/camera.h`, `scene/integrator.h`, `scene/scene.h`, `session/session.h`
  - 条件编译: `hydra/display_driver.h`（`WITH_HYDRA_DISPLAY_DRIVER`）
- **外部依赖**: `pxr/imaging/hd/renderPass.h`, `pxr/imaging/hd/renderPassState.h`
- **被引用**: `render_delegate.cpp`

## 实现细节 / 关键算法

1. **构造与析构**: 构造时重置会话进度、设置输出驱动（`HdCyclesOutputDriver`），并根据显示支持情况设置显示驱动（`HdCyclesDisplayDriver`）。析构时调用 `session->cancel(true)` 取消渲染。

2. **`_Execute` 执行流程**:
   - 检查渲染是否被取消，若取消则直接返回
   - 尝试获取场景互斥锁（`try_lock`），获取失败则跳过本帧
   - 同步 AOV 绑定（当绑定列表变化或去噪设置修改时）
   - 更新显示 AOV 绑定（仅在支持显示驱动时，将第一个 color AOV 设为显示通道）
   - 从 `HdRenderPassState` 获取视口/帧窗口尺寸，更新相机的 `full_width` / `full_height`
   - 应用相机设置（支持 `HdCyclesCamera` 对象或矩阵参数两种路径）
   - 当场景需要重置或设置版本号变化时，重置所有渲染缓冲区的收敛状态，构建 `BufferParams` 并调用 `session->reset()`
   - 释放场景锁后调用 `session->start()` 启动渲染线程
   - 最后调用 `session->draw()` 进行绘制

3. **版本兼容**: 通过 `PXR_VERSION` 宏区分 USD 2102 前后的帧窗口（`CameraUtilFraming`）API。

4. **收敛判定**: 遍历所有 AOV 绑定的渲染缓冲区，全部收敛时返回 `true`。

## 关联文件

- `hydra/render_delegate.h` -- 创建本渲染通道
- `hydra/render_buffer.h` -- 渲染缓冲区收敛状态管理
- `hydra/output_driver.h` -- 像素输出驱动
- `hydra/display_driver.h` -- 实时显示驱动（可选）
- `hydra/camera.h` -- 相机参数应用
- `hydra/session.h` -- 会话管理与 AOV 绑定
