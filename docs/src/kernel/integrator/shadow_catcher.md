# shadow_catcher.h - 阴影捕捉器路径分裂逻辑

## 概述

`shadow_catcher.h` 实现了阴影捕捉器(Shadow Catcher)功能的核心判断逻辑。阴影捕捉器是一种特殊对象，用于在合成工作流中捕捉来自其他物体的阴影投射，使其能够与背景素材无缝合成。该文件提供了路径分裂判断、路径类型识别等工具函数，是实现阴影捕捉器渲染通道分离的基础。

## 核心函数

### `kernel_shadow_catcher_is_path_split_bounce`
判断当前表面弹射点是否应该触发阴影捕捉器路径分裂。条件包括：
- 场景中存在阴影捕捉器
- 当前对象标记为阴影捕捉器（`SD_OBJECT_SHADOW_CATCHER`）
- 对象不是 holdout 遮罩
- 路径为主光线（具有透明背景标志）
- 尚未命中过阴影捕捉器

### `kernel_shadow_catcher_path_can_split`
检查当前路径是否仍可进行分裂。已终止的路径或已命中阴影捕捉器的路径不可再分裂。

### `kernel_shadow_catcher_is_matte_path`
判断当前路径是否为磨砂（matte）路径，即尚未命中阴影捕捉器的路径。

### `kernel_shadow_catcher_is_object_pass`
判断当前路径是否为阴影捕捉器的对象通道。

## 依赖关系

- **内部头文件**: `kernel/integrator/state_flow.h` - 路径状态控制流
- **被引用**: `intersect_closest.h`（交点判断时检查分裂条件）、`kernel/film/light_passes.h`（写入通道数据时区分路径类型）

## 实现细节 / 关键算法

1. **路径分裂机制**: 当主光线首次命中阴影捕捉器对象时，路径被分裂为两条：一条继续追踪阴影捕捉器表面（记录阴影信息），另一条标记为 `PATH_RAY_SHADOW_CATCHER_PASS` 继续追踪场景其余部分。这种分裂确保最终渲染输出可以正确合成到实拍背景上。

2. **条件判断层级**: 函数采用早期返回模式，按开销从低到高依次检查：先检查路径标志（寄存器操作），再检查对象标志（可能涉及全局内存读取），避免不必要的内存访问。

3. **仅主光线分裂**: 分裂仅在主光线（`PATH_RAY_TRANSPARENT_BACKGROUND`）上发生，二次弹射将阴影捕捉器视为普通对象处理，避免路径指数增长。

## 关联文件

- `state_flow.h` - 提供路径终止检查函数
- `intersect_closest.h` - 在最近交点内核中调用分裂判断
- `state_util.h` - 提供 `integrator_state_shadow_catcher_split` 执行实际的状态分裂
