# queue.h - 队列容器类型别名封装

## 概述

`queue.h` 将标准库中的 `std::queue` 引入到 Cycles 命名空间中。这是一个极简的封装文件，使项目代码可以直接使用 `queue` 而无需 `std::` 前缀。`queue` 是先进先出（FIFO）容器适配器，默认底层使用 `std::deque` 实现。

## 类与结构体

该文件不定义新的类或结构体，仅通过 `using` 声明引入标准库类型：

- `queue` -- `std::queue`（先进先出队列容器适配器）

## 核心函数

无。该文件仅包含类型别名声明。

## 依赖关系

- **内部头文件**: 无
- **标准库**: `<queue>`
- **被引用**: `scene/shader_graph.cpp`

## 关联文件

- `util/deque.h` -- `std::queue` 默认底层容器 `std::deque` 的封装
- `util/list.h` -- 链表容器封装
