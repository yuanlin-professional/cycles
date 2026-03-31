# tables.h / tables.cpp - 查找表管理器

## 概述

本文件实现了 Cycles 渲染器的查找表（Lookup Tables）管理系统 `LookupTables`。查找表用于存储着色器所需的预计算数据（如渐变映射曲线、颜色渐变等），管理器负责在设备端全局查找表缓冲区中分配、管理和回收空间。采用首次适配内存分配策略，以固定块大小（256）对齐。

## 类与结构体

### LookupTables
- **功能**: 管理设备端全局查找表缓冲区的分配与释放
- **关键成员**:
  - `lookup_tables` — 已分配表的链表（`list<Table>`），按偏移量有序排列
  - `need_update_` — 更新标记
- **内部结构 Table**:
  - `offset` — 在全局缓冲区中的起始偏移量
  - `size` — 分配的大小（对齐到 TABLE_CHUNK_SIZE）
- **关键方法**:
  - `add_table()` — 在全局查找表缓冲区中分配空间，复制数据并返回偏移量
  - `remove_table()` — 通过偏移量移除已分配的表
  - `device_update()` — 将查找表数据拷贝到设备
  - `device_free()` — 释放设备端查找表内存
  - `need_update()` — 检查是否需要设备更新

## 核心函数

- `round_up_to_multiple()` — 内部辅助函数，将大小向上取整到指定块的倍数

## 常量

- `TABLE_CHUNK_SIZE = 256` — 查找表分配的最小对齐块大小
- `TABLE_OFFSET_INVALID = -1` — 无效偏移量标记

## 依赖关系

- **内部头文件**: `util/list.h`, `util/vector.h`
- **cpp 额外引用**: `device/device.h`, `scene/scene.h`, `scene/stats.h`, `util/log.h`, `util/time.h`
- **被引用**: `scene/scene.cpp`（Scene 持有 `LookupTables`）, `scene/film.cpp`, `scene/camera.cpp`, `scene/shader.cpp`

## 实现细节 / 关键算法

1. **首次适配分配**: `add_table()` 遍历已有的有序链表，寻找第一个能容纳新表的空隙（`new_table.offset + new_table.size <= table->offset`）。如果没有合适空隙，追加到末尾并调整设备缓冲区大小。
2. **块对齐**: 所有分配的大小通过 `round_up_to_multiple()` 向上对齐到 `TABLE_CHUNK_SIZE`（256），减少碎片化。
3. **懒更新**: `device_update()` 检查 `need_update_` 标志，仅在有变更时执行设备数据拷贝。
4. **链表管理**: 使用 `std::list` 而非 `vector` 存储表条目，便于中间插入和删除操作。

## 关联文件

- `scene/scene.h/.cpp` — 场景管理（持有 `LookupTables` 实例，在 `device_update()` 中调用两次——几何阶段和胶片阶段各一次）
- `scene/film.cpp` — 胶片（使用查找表存储 mist/颜色管理曲线）
- `scene/camera.cpp` — 相机（可能使用查找表存储镜头畸变数据）
- `scene/shader.cpp` — 着色器（使用查找表存储渐变映射数据）
- `scene/devicescene.h` — 设备端场景（`lookup_table` 设备向量）
