# concurrent_set.h - 线程安全集合容器封装

## 概述

`concurrent_set.h` 将 Intel TBB（Threading Building Blocks）库中的 `tbb::concurrent_set` 引入到 Cycles 命名空间中。`concurrent_set` 是一个支持并发插入和查找操作的有序集合容器，适用于多线程环境下无需显式加锁的集合操作场景。在 Windows 平台上，该文件先行包含 `util/windows.h` 以确保 `WIN32_LEAN_AND_MEAN` 等宏在 TBB 包含 `<windows.h>` 之前已经定义。

## 类与结构体

该文件不定义新的类或结构体，仅通过 `using` 声明引入 TBB 类型：

- `concurrent_set` -- `tbb::concurrent_set`（线程安全的有序集合容器，支持并发插入、查找和迭代）

## 核心函数

无。该文件仅包含类型别名声明。

## 依赖关系

- **内部头文件**: `util/windows.h`（仅 Windows 平台，`_WIN32` 宏下）
- **外部库**: Intel TBB（`<tbb/concurrent_set.h>`）
- **被引用**: 当前代码库中未发现直接引用该头文件的文件（可能用于特定构建配置或插件扩展）

## 关联文件

- `util/set.h` -- 非线程安全的标准集合容器封装
- `util/concurrent_vector.h` -- 同样基于 TBB 的线程安全向量容器封装
- `util/windows.h` -- Windows 平台头文件预处理，确保 `WIN32_LEAN_AND_MEAN` 等宏的正确定义
