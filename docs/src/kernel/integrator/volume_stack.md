# volume_stack.h - 体积栈管理与采样方法选择

## 概述

`volume_stack.h` 管理体积栈的进出操作和体积属性查询。体积栈是一个对象/着色器ID数组，记录当前路径段穿越的所有体积对象。该文件提供了体积栈的读写、对象进入/退出、均匀性检查、步长计算、采样方法选择等功能，是体积渲染中追踪光线所处介质的核心基础设施。

## 核心函数

### 栈读写
- **`volume_stack_read<shadow>`**: 模板函数，根据是否为阴影路径选择读取主路径体积栈或阴影体积栈。
- **`volume_stack_write<shadow>`**: 模板函数，向对应的体积栈写入条目。

### 栈进出
- **`volume_stack_enter_exit<shadow>`**: 处理光线穿过体积表面时的栈更新。当光线从背面射出时（`SD_BACKFACING`），从栈中移除对应条目并后移后续条目；当光线从正面射入时，将对象/着色器追加到栈尾。跳过非体积对象和已存在于栈中的重复条目。栈满时忽略新条目。

### 栈清理
- **`volume_stack_clean`**: 在最后一次弹射后清理体积栈，仅保留世界体积。解决因光线交点精度问题导致的非世界体积残留在栈中的伪影。

### 均匀性检查
- **`volume_is_homogeneous`** (单条目版): 检查单个体积栈条目是否为均匀体积。非均匀条件：着色器标记为 `SD_HETEROGENEOUS_VOLUME`，或同时需要体积属性且对象具有体积属性。
- **`volume_is_homogeneous<shadow>`** (全栈版): 遍历栈中所有条目，只要有一个非均匀即返回 false。

### 步长计算
- **`volume_stack_step_size<shadow>`**: 计算光线行进步长。遍历栈中非均匀体积，返回所有对象步长的最小值。

### 采样方法选择
- **`volume_stack_sample_method`**: 确定体积采样方法。遍历栈中所有条目的着色器标志：
  - `SD_VOLUME_MIS`: 直接返回多重重要性采样
  - `SD_VOLUME_EQUIANGULAR`: 等角采样
  - 其他: 距离采样
  - 当混合出现距离采样和等角采样时，升级为 MIS

## 依赖关系

- **内部头文件**: 无直接 #include（仅使用 CCL_NAMESPACE 宏和在其他文件中已包含的类型）
- **被引用**: `shade_surface.h`, `shade_volume.h`, `shade_shadow.h`, `volume_shader.h`, `intersect_volume_stack.h`

## 实现细节 / 关键算法

1. **栈进出的线性搜索**: 进入和退出操作都使用线性搜索遍历栈。退出时找到匹配条目后，通过循环后移操作压缩栈。进入时在栈尾追加并写入 SHADER_NONE 哨兵。栈大小由 `kernel_data.volume_stack_size` 限制。

2. **重复条目检测**: 进入体积时如果对象和着色器已在栈中，直接返回不做任何操作。这处理了光线精度问题导致的同一表面多次进入场景。

3. **世界体积保留**: `volume_stack_clean` 特殊处理世界体积：如果存在世界体积着色器，保留栈的第一个条目（假定为世界体积），仅清除后续条目。

4. **均匀性判断逻辑**: 均匀性不仅取决于着色器标志，还取决于对象是否实际具有体积属性（`SD_OBJECT_HAS_VOLUME_ATTRIBUTES`）。世界体积不支持体积属性，因此始终被视为均匀。

5. **采样方法优先级**: MIS采样的优先级最高。当栈中存在不同采样需求的体积时（一个需要距离采样，另一个需要等角采样），自动升级为MIS以确保两种方法都被考虑。

6. **Apple GPU 优化**: 在使用数据常量的平台（Apple GPU）上，当场景不包含体积特性时，`volume_stack_enter_exit` 提前返回，编译器可以死代码消除整个函数，提升1-2%性能。

## 关联文件

- `shade_volume.h` - 体积着色流程中使用栈数据
- `shade_surface.h` - 表面弹射后更新体积栈
- `shade_shadow.h` - 阴影光线使用阴影体积栈
- `volume_shader.h` - 体积着色器评估时遍历栈
- `state_util.h` - 体积栈的状态读写底层实现
- `state_template.h` - 定义 `volume_stack` 数据结构
