# memory.h / memory.cpp - 设备内存管理抽象层

## 概述

本文件是 Cycles 渲染器设备内存管理的核心模块，定义了用于在主机与设备之间分配、拷贝和释放内存的完整类型体系。它包含基类 `device_memory` 及其四个主要子类：`device_only_memory`（仅设备内存）、`device_vector`（主机-设备数据交换向量）、`device_sub_ptr`（子内存指针）和 `device_texture`（纹理内存），并通过模板特化的 `device_type_traits` 实现类型安全的数据格式描述。

## 类与结构体

### device_memory
- **继承**: 无（基类）
- **功能**: 所有设备内存的抽象基类，封装主机/设备指针管理、内存分配/释放/拷贝的统一接口。禁止拷贝和移动，以确保设备端分配映射表的一致性。
- **关键成员**:
  - `data_type` — 数据类型枚举（`DataType`）
  - `data_elements` — 每个元素包含的分量数
  - `data_size` — 元素总数
  - `device_size` — 设备端实际分配大小（字节）
  - `data_width` / `data_height` — 2D 数据的宽高
  - `type` — 内存类型（`MemoryType`）
  - `name` — 内存块名称（用于调试）
  - `device` — 所属设备指针
  - `device_pointer` — 设备端指针
  - `host_pointer` — 主机端指针
  - `shared_pointer` — 共享内存指针（主机映射模式）
  - `shared_counter` — 共享内存引用计数
  - `move_to_host` — 标记是否正在迁移至主机
- **关键方法**:
  - `memory_size()` — 计算总内存大小（字节）
  - `host_alloc(size)` — 在设备关联的主机端分配对齐内存
  - `device_alloc()` — 在设备端分配内存
  - `device_copy_to()` — 从主机拷贝到设备
  - `device_copy_from(y, w, h, elem)` — 从设备拷贝到主机（支持区域拷贝）
  - `device_zero()` — 设备端内存清零
  - `host_and_device_free()` — 同时释放主机和设备内存
  - `swap_device()` / `restore_device()` — 临时切换/恢复设备关联（多设备场景）
  - `is_resident(sub_device)` — 检查内存是否驻留在指定子设备上
  - `is_shared(sub_device)` — 检查是否为共享内存模式

### device_only_memory\<T\>
- **继承**: `device_memory`
- **功能**: 仅在设备端分配的工作内存，主机端不分配对应缓冲区。主要用于设备内部的临时工作缓冲区。
- **关键方法**:
  - `alloc_to_device(num, shrink_to_fit)` — 在设备端分配指定数量的元素，支持 `shrink_to_fit` 控制是否在缩小时重新分配
  - `free()` — 释放设备内存
  - `zero_to_device()` — 设备端清零

### device_vector\<T\>
- **继承**: `device_memory`
- **功能**: 主机-设备数据交换的核心容器。在主机端先分配和填充数据，然后拷贝到设备端。支持 `MEM_GLOBAL` 类型自动绑定到内核全局数据。
- **关键方法**:
  - `alloc(width, height)` — 在主机端分配内存
  - `resize(width, height)` — 调整大小并保留已有数据
  - `steal_data(array)` — 从 `array<T>` 接管数据所有权
  - `free()` — 释放主机和设备内存
  - `copy_to_device()` — 拷贝到设备
  - `copy_to_device_if_modified()` — 仅在修改后拷贝
  - `copy_from_device()` — 从设备拷回
  - `zero_to_device()` — 设备端清零
  - `tag_modified()` / `tag_realloc()` — 标记修改/需要重新分配
  - `data()` — 返回主机端数据指针
  - `operator[]` — 下标访问

### device_sub_ptr
- **继承**: 无
- **功能**: 指向已有 `device_memory` 的一个子区域的设备指针。不独立分配内存，在其生命周期结束时自动释放子指针。
- **关键方法**:
  - 构造函数 `device_sub_ptr(mem, offset, size)` — 从已有内存中创建偏移子指针
  - `operator*()` — 解引用获取 `device_ptr`

### device_texture
- **继承**: `device_memory`
- **功能**: 2D 纹理内存，用于图像纹理数据的设备端管理。支持多种图像数据格式（float4、byte4、half4 等）和纹理采样属性。
- **关键成员**:
  - `slot` — 纹理槽位编号
  - `info` — 纹理元信息（`TextureInfo`），包含数据类型、插值模式、扩展模式
- **关键方法**:
  - `alloc(width, height)` — 分配纹理主机内存
  - `copy_to_device()` — 拷贝纹理到设备

## 核心函数

- `datatype_size(DataType)` — constexpr 函数，返回指定数据类型的字节大小。
- `device_memory::host_alloc()` — 通过 `device->host_alloc()` 分配对齐内存，分配失败时抛出 `std::bad_alloc`。
- `device_memory::host_and_device_free()` — 先释放主机内存（跳过共享指针），再通过 `device->mem_free()` 释放设备内存。

## 枚举类型

### MemoryType
- `MEM_READ_ONLY` — 只读内存
- `MEM_READ_WRITE` — 读写内存
- `MEM_DEVICE_ONLY` — 仅设备内存
- `MEM_GLOBAL` — 全局内存（自动绑定到内核全局数据）
- `MEM_TEXTURE` — 纹理内存

### DataType
- `TYPE_UNKNOWN`、`TYPE_UCHAR`、`TYPE_UINT16`、`TYPE_UINT`、`TYPE_INT`、`TYPE_FLOAT`、`TYPE_HALF`、`TYPE_UINT64`

## 模板特化 device_type_traits

为以下类型提供了 `data_type` 和 `num_elements` 编译期常量：
- 标量: `uchar`、`uint`、`int`、`float`、`half`、`uint16_t`、`uint64_t`
- 向量: `uchar2`/`3`/`4`、`uint2`/`4`、`int2`/`4`、`float2`/`4`、`half4`、`ushort4`、`packed_float3`
- **注意**: `uint3`、`int3`、`float3` 故意留空未特化，因为它们在不同设备上大小不一致，使用会触发编译错误

## 依赖关系

- **内部头文件**: `util/array.h`、`util/half.h`、`util/string.h`、`util/texture.h`、`util/types.h`
- **实现文件额外依赖**: `device/device.h`
- **外部库**: 无
- **被引用**:
  - `src/device/device.h` — 设备基类引用内存类型
  - `src/session/buffers.h` — 渲染缓冲区使用 `device_vector`
  - `src/scene/devicescene.h` / `.cpp` — 场景数据的设备端管理
  - `src/scene/image.h` — 图像管理使用纹理内存
  - `src/integrator/path_trace_work_gpu.h` — GPU 路径追踪工作区内存管理
  - `src/integrator/shader_eval.h` — 着色器求值缓冲区
  - `src/integrator/denoiser_gpu.cpp` — 降噪器 GPU 内存管理
  - `src/device/cpu/device_impl.h` — CPU 设备内存实现

## 实现细节 / 关键算法

- **禁止拷贝/移动**: `device_memory` 显式删除了拷贝构造、移动构造和赋值运算符。这是因为设备后端可能使用内存对象指针作为分配映射表的键，移动会导致映射表失效。
- **共享内存管理**: `shared_pointer` 和 `shared_counter` 用于多设备共享同一主机映射内存的场景。`host_and_device_free()` 在释放主机指针时会检查是否与 `shared_pointer` 相同，避免双重释放。
- **设备切换**: `swap_device()` 和 `restore_device()` 将原始设备信息保存到 `original_device*` 字段中，用于多设备渲染时临时将内存关联到另一个设备。
- **纹理格式映射**: `device_texture` 构造函数中根据 `ImageDataType` 设置对应的 `data_type` 和 `data_elements`，支持 float/byte/half/ushort 的 1/4 分量格式以及多种 NanoVDB 格式。
- **resize 保留数据**: `device_vector::resize()` 在重新分配时，先分配新缓冲区，逐元素拷贝旧数据，新增区域用默认构造的 `T()` 填充。

## 关联文件

- `src/device/device.h` — `Device` 基类是 `device_memory` 的友元类，提供底层内存操作
- `src/device/queue.h` — 命令队列提供异步内存拷贝接口
- `src/scene/devicescene.h` — 使用 `device_vector` 管理场景数据的设备副本
- `src/session/buffers.h` — 渲染缓冲区基于 `device_vector` 实现
- `src/util/texture.h` — 提供 `TextureInfo` 结构体定义
