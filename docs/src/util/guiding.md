# guiding.h - 路径引导库集成封装

## 概述
本文件封装了 Intel Open Path Guiding Library（OpenPGL）的头文件引入和平台支持检测。路径引导是一种通过学习场景中的光传输分布来优化蒙特卡洛采样策略的技术，可显著提高复杂光照场景的收敛速度。本文件通过编译条件 `WITH_PATH_GUIDING` 控制是否启用，并提供运行时 CPU/架构能力检测。

## 类与结构体
无自定义类或结构体。当 `WITH_PATH_GUIDING` 启用时，导出 `<openpgl/cpp/OpenPGL.h>` 和 `<openpgl/version.h>` 中的 OpenPGL API。

## 核心函数

### `int guiding_device_type()`
检测并返回路径引导的设备类型（SIMD 宽度）：

| 条件 | 返回值 | 说明 |
|------|--------|------|
| ARM NEON | 8 | ARM 架构 NEON 指令集 |
| x86 AVX2 | 8 | 8 宽度 SIMD |
| x86 SSE4.2 | 4 | 4 宽度 SIMD |
| 不满足最低要求 | 0 | 不支持 |
| 未编译 OpenPGL | 0 | 不支持 |

### `bool guiding_supported()`
检测是否支持路径引导功能，等价于 `guiding_device_type() != 0`。

## 依赖关系
- **内部头文件**: `util/system.h`（CPU 能力检测）
- **外部依赖**: `<openpgl/cpp/OpenPGL.h>`, `<openpgl/version.h>`（条件编译）
- **编译条件**: `WITH_PATH_GUIDING`
- **被引用**: `integrator/path_state.h`, `integrator/guiding.h`, `session/session.cpp`

## 关联文件
- `util/system.h` - `system_cpu_support_sse42()`, `system_cpu_support_avx2()` CPU 特性检测
- `integrator/guiding.h` - 路径引导积分器层面的接口
