# list.h - 双向链表类型别名封装

## 概述

`list.h` 将标准库中的 `std::list` 引入到 Cycles 命名空间中。这是一个极简的封装文件，使项目代码可以直接使用 `list` 而无需 `std::` 前缀。`list` 是双向链表容器，支持任意位置的常数时间插入和删除操作。

## 类与结构体

该文件不定义新的类或结构体，仅通过 `using` 声明引入标准库类型：

- `list` -- `std::list`（双向链表容器）

## 核心函数

无。该文件仅包含类型别名声明。

## 依赖关系

- **内部头文件**: 无
- **标准库**: `<list>`
- **被引用**: `util/task.h`, `scene/tables.h`, `scene/attribute.h`, `device/multi/device.cpp`

## 关联文件

- `util/vector.h` -- 动态数组容器封装
- `util/deque.h` -- 双端队列容器封装
