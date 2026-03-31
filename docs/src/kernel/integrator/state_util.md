# state_util.h - 积分器状态读写与复制工具

## 概述

`state_util.h` 提供了积分器状态数据的读写、复制和移动工具函数。它封装了光线、交点、体积栈、阴影光线等数据在积分器状态和局部变量之间的序列化/反序列化操作，并实现了 GPU 上路径状态的完整复制以及阴影捕捉器的状态分裂。该文件还提供了跨 CPU/GPU 统一接口的弹射深度查询函数。

## 核心函数

### 光线读写
- **`integrator_state_write_ray`**: 将 `Ray` 结构体写入积分器状态。GPU 打包模式下将 `dP`/`dD` 嵌入 `float3` 的第四分量进行单次写入。
- **`integrator_state_read_ray`**: 从积分器状态读取 `Ray`。GPU 打包模式下从打包结构一次性读取后解包。

### 阴影光线读写
- **`integrator_state_write_shadow_ray`**: 写入阴影光线数据。
- **`integrator_state_read_shadow_ray`**: 读取阴影光线数据，`dD` 设为零（阴影光线不需要方向微分）。
- **`integrator_state_write_shadow_ray_self`**: 写入阴影光线的自交测试数据，使用 `shadow_isect` 数组的最后一个元素存储 `self.object`/`self.prim`，专用字段存储 `self_light_object`/`self_light_prim`。
- **`integrator_state_read_shadow_ray_self`**: 读取阴影光线自交测试数据。

### 交点读写
- **`integrator_state_write_isect`**: 写入场景交点。GPU 打包模式下使用 `packed_isect` 单次写入。
- **`integrator_state_read_isect`**: 读取场景交点。

### 体积栈操作
- **`integrator_state_read_volume_stack`** / **`integrator_state_write_volume_stack`**: 读写主路径体积栈条目。
- **`integrator_state_volume_stack_is_empty`**: 检查体积栈是否为空。
- **`integrator_state_copy_volume_stack_to_shadow`**: 将主路径体积栈复制到阴影路径。
- **`integrator_state_copy_volume_stack`**: 在两个主路径间复制体积栈。
- **`integrator_state_read_shadow_volume_stack`** / **`integrator_state_write_shadow_volume_stack`**: 阴影路径体积栈读写。
- **`integrator_state_shadow_volume_stack_is_empty`**: 检查阴影体积栈是否为空。

### 阴影交点读写
- **`integrator_state_write_shadow_isect`**: 按索引写入阴影交点。
- **`integrator_state_read_shadow_isect`**: 按索引读取阴影交点。

### 状态复制与移动（仅GPU）
- **`integrator_state_copy_only`**: 通过宏模板展开复制主路径的全部状态字段。
- **`integrator_state_move`**: 复制后将源路径标记为终止。
- **`integrator_shadow_state_copy_only`** / **`integrator_shadow_state_move`**: 阴影路径的复制和移动。

### 阴影捕捉器分裂
- **`integrator_state_shadow_catcher_split`**: 为阴影捕捉器创建路径副本。GPU 上从全局索引池原子分配新路径；CPU 上使用 `state + 1` 指针并仅复制必要子集以优化性能。

### 弹射深度查询
- **`integrator_state_bounce`** / **`integrator_state_diffuse_bounce`** / **`integrator_state_glossy_bounce`** / **`integrator_state_transmission_bounce`** / **`integrator_state_transparent_bounce`** / **`integrator_state_portal_bounce`**: 统一查询接口，CPU 上通过重载区分主路径和阴影路径，GPU 上通过 `path_flag` 中的 `PATH_RAY_SHADOW` 位区分。

## 依赖关系

- **内部头文件**:
  - `kernel/globals.h` - 全局内核数据
  - `kernel/integrator/state.h` - 状态数据结构与访问宏
  - `kernel/util/differential.h` - 微分工具
- **被引用**: `init_from_camera.h`（初始化路径）、`intersect_shadow.h`（阴影交点处理）、GPU/OptiX/CPU 内核文件

## 实现细节 / 关键算法

1. **GPU 打包布局优化**: 光线和交点数据在 GPU 打包模式下使用 `packed_ray`/`packed_isect` 结构进行整体读写。利用 `float3` 的 16 字节对齐填充位存储 `dP`/`dD`，通过 `static_assert` 验证偏移一致性。

2. **阴影光线自交保持**: `write_shadow_ray_self` 的复杂逻辑源于阴影光线可能被多次调用 `intersect_shadow` 内核。`self.object`/`self.prim` 存储在 `shadow_isect` 数组尾部以在后续调用中携带最远交点信息，`self_light_object`/`self_light_prim` 使用专用字段确保灯光自交信息不被覆盖。

3. **阴影捕捉器分裂的CPU优化**: CPU 上仅复制 `path`、`ray`、`isect` 和体积栈，跳过其他状态字段，因为 CPU 的阴影捕捉器分裂路径紧接着就会被处理。

4. **弹射深度的CPU/GPU接口差异**: CPU 上利用 C++ 函数重载区分 `IntegratorState` 和 `IntegratorShadowState`，GPU 上由于两者都是 `int` 类型无法重载，改为传入 `path_flag` 参数在运行时判断。

## 关联文件

- `state.h` - 提供基础数据结构和访问宏
- `state_template.h` / `shadow_state_template.h` - 用于生成状态复制代码
- `state_flow.h` - 路径调度和终止
- `path_state.h` - 更高层的状态操作
