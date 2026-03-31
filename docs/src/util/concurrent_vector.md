# concurrent_vector.h - 线程安全向量容器封装

## 概述

`concurrent_vector.h` 将 Intel TBB（Threading Building Blocks）库中的 `tbb::concurrent_vector` 引入到 Cycles 命名空间中。`concurrent_vector` 是一个支持并发增长（grow）操作的动态数组，多个线程可以同时向其追加元素而无需加锁。在 Windows 平台上，该文件先行包含 `util/windows.h` 以确保 `WIN32_LEAN_AND_MEAN` 等宏在 TBB 包含 `<windows.h>` 之前已经定义。

## 类与结构体

该文件不定义新的类或结构体，仅通过 `using` 声明引入 TBB 类型：

- `concurrent_vector` -- `tbb::concurrent_vector`（线程安全的动态数组，支持并发追加，已有元素的地址在增长后保持稳定）

## 核心函数

无。该文件仅包含类型别名声明。

## 依赖关系

- **内部头文件**: `util/windows.h`（仅 Windows 平台，`_WIN32` 宏下）
- **外部库**: Intel TBB（`<tbb/concurrent_vector.h>`）
- **被引用**: 当前代码库中未发现直接引用该头文件的文件（可能用于特定构建配置或插件扩展）

## 关联文件

- `util/vector.h` -- 非线程安全的标准向量容器封装
- `util/concurrent_set.h` -- 同样基于 TBB 的线程安全集合容器封装
- `util/windows.h` -- Windows 平台头文件预处理
