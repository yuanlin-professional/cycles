# unique_ptr.h - 独占指针类型别名封装

## 概述

`unique_ptr.h` 将标准库中的 `std::unique_ptr` 和 `std::make_unique` 引入到 Cycles 命名空间中。这是一个极简的封装文件，使项目代码可以直接使用 `unique_ptr` 和 `make_unique` 而无需 `std::` 前缀，保持与其他 Cycles 工具类型一致的命名风格。

## 类与结构体

该文件不定义新的类或结构体，仅通过 `using` 声明引入标准库类型：

- `unique_ptr` -- `std::unique_ptr`（独占所有权的智能指针）
- `make_unique` -- `std::make_unique`（安全构造 `unique_ptr` 的工厂函数）

## 核心函数

无。该文件仅包含类型别名声明。

## 依赖关系

- **内部头文件**: 无
- **标准库**: `<memory>`
- **被引用**: `util/unique_ptr_vector.h`, `util/task.h`, `util/profiling.h`, `session/` 目录多个文件, `scene/` 目录多个文件, `integrator/` 目录多个文件, `device/` 目录大量文件, `bvh/` 目录多个文件等，共计 39 个以上文件引用

## 关联文件

- `util/unique_ptr_vector.h` -- 基于 `unique_ptr` 构建的便捷容器
- `util/vector.h` -- 常与 `unique_ptr` 组合使用（`vector<unique_ptr<T>>`）
