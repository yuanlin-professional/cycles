# state.h - 积分器状态数据结构与抽象访问宏

## 概述

`state.h` 是 Cycles 积分器状态系统的核心文件，定义了路径追踪过程中所有需要在内核执行之间保存和传递的状态数据结构。该文件通过宏抽象层实现了 CPU（标量AoS布局）和 GPU（批量SoA布局）之间的统一访问接口，使上层代码无需关心底层内存布局差异。所有积分器内核都依赖此文件中定义的状态访问宏。

## 核心函数

本文件主要定义数据结构和宏，不包含算法函数。

### 数据结构

#### `IntegratorShadowStateCPU`
CPU 端阴影路径状态结构体，通过包含 `shadow_state_template.h` 并设置宏生成 AoS 布局的嵌套结构体。

#### `IntegratorStateCPU`
CPU 端主路径状态结构体，通过包含 `state_template.h` 生成，额外包含 `shadow`（阴影状态）和 `ao`（环境光遮蔽状态）两个 `IntegratorShadowStateCPU` 成员。

#### `IntegratorQueueCounter`
GPU 路径队列计数器，维护每个内核类型排队的路径数量。

#### `IntegratorStateGPU`
GPU 端状态结构体，使用 SoA 布局，每个字段为指向全局内存数组的指针。支持可选的打包布局（`__INTEGRATOR_GPU_PACKED_STATE__`）。额外包含队列计数器、排序键计数器、阴影路径索引等 GPU 调度辅助字段。

### 访问宏

- **`INTEGRATOR_STATE(state, x, y)`**: 读取嵌套结构成员 `x.y`
- **`INTEGRATOR_STATE_WRITE(state, x, y)`**: 写入嵌套结构成员 `x.y`
- **`INTEGRATOR_STATE_ARRAY(state, x, i, y)`**: 读取数组结构 `x[i].y`
- **`INTEGRATOR_STATE_ARRAY_WRITE(state, x, i, y)`**: 写入数组结构 `x[i].y`
- **`INTEGRATOR_STATE_NULL`**: 空状态（CPU 为 `nullptr`，GPU 为 `-1`）

### 类型别名

- CPU: `IntegratorState = IntegratorStateCPU*`, `IntegratorShadowState = IntegratorShadowStateCPU*`
- GPU: `IntegratorState = int`（路径索引），`IntegratorShadowState = int`

## 依赖关系

- **内部头文件**:
  - `kernel/types.h` - 基础类型定义
  - `util/types.h` - 工具类型
  - `util/guiding.h` - 路径引导（OpenPGL，条件编译）
  - `kernel/integrator/state_template.h` - 主路径状态模板（通过宏包含）
  - `kernel/integrator/shadow_state_template.h` - 阴影路径状态模板（通过宏包含）
- **被引用**: 几乎所有积分器内核文件，通过 `state_flow.h`、`state_util.h`、`path_state.h` 等间接引用。直接引用者包括 `state_flow.h`, `state_util.h`, `guiding.h`, `intersect_shadow.h`, `path_state.h` 以及各平台 globals 头文件。

## 实现细节 / 关键算法

1. **宏抽象层设计**: 同一套 `INTEGRATOR_STATE` 宏在 CPU 和 GPU 上展开为完全不同的代码。CPU 上直接解引用指针访问结构体成员，GPU 上通过路径索引访问全局数组。这使得所有上层内核代码可以完全共享。

2. **GPU 打包布局**: 当启用 `__INTEGRATOR_GPU_PACKED_STATE__` 时，将相关字段（如光线的 P、D、tmin、tmax）打包到连续内存中，通过 `packed_ray` 等包装结构和生成的 `member_fn()` 访问器函数实现透明访问，减少 GPU 全局内存事务。

3. **CPU 双阴影状态**: CPU 端的 `IntegratorStateCPU` 包含两个阴影状态（`shadow` 和 `ao`），因为 CPU 上阴影路径在主路径内同步执行，同一时刻可能需要普通阴影和AO阴影两个状态。GPU 上阴影路径通过全局索引池独立分配。

4. **GPU 排序支持**: `IntegratorStateGPU` 包含 `sort_key_counter` 和 `sort_partition_key_offsets` 等字段，用于按材质/着色器对路径排序以提升 GPU 缓存一致性。`sort_partition_divisor` 将状态索引分区以在排序中同时兼顾空间局部性。

## 关联文件

- `state_template.h` - 主路径状态模板定义
- `shadow_state_template.h` - 阴影路径状态模板定义
- `state_flow.h` - 基于状态的内核控制流
- `state_util.h` - 状态读写和复制工具
