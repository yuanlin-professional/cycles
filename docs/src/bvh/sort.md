# sort.h / sort.cpp - 层次包围体(BVH)图元引用的多线程排序

## 概述

本模块实现了 BVH 构建过程中图元引用（`BVHReference`）的排序功能。排序按照图元包围盒在指定轴上的中心坐标进行，支持非对齐空间变换。对于大规模数据集采用多线程快速排序（基于中位数三点法的并行 QSort），小数据集则直接使用标准排序算法。排序是 BVH 构建中选择最优分割平面的关键步骤。

## 类与结构体

### `BVHReferenceCompare` (内部结构体)
图元引用的比较器，用于排序过程。

| 成员 | 类型 | 说明 |
|------|------|------|
| `dim` | `int` | 排序维度（0=X, 1=Y, 2=Z） |
| `unaligned_heuristic` | `const BVHUnaligned *` | 非对齐启发式（可为空） |
| `aligned_space` | `const Transform *` | 对齐空间变换矩阵（可为空） |

**关键方法:**

| 方法 | 说明 |
|------|------|
| `get_prim_bounds(prim)` | 获取图元包围盒，如果存在对齐空间则计算变换后的包围盒 |
| `compare(ra, rb)` | 比较两个引用，返回类似 `strcmp` 的值（负/零/正） |
| `operator()(ra, rb)` | 排序操作符，返回 `compare < 0` |

**比较规则（优先级从高到低）:**
1. 包围盒在 `dim` 轴上的中心坐标（`min[dim] + max[dim]`）
2. 对象索引（`prim_object`）
3. 图元索引（`prim_index`）
4. 图元类型（`prim_type`）

## 核心函数

### `bvh_reference_sort(start, end, data, dim, unaligned_heuristic, aligned_space)`
主排序入口函数（声明在 sort.h 中，定义在 sort.cpp 中）。

**参数:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `start` | `int` | 排序起始索引 |
| `end` | `int` | 排序结束索引（不包含） |
| `data` | `BVHReference *` | 图元引用数组 |
| `dim` | `int` | 排序维度 |
| `unaligned_heuristic` | `const BVHUnaligned *` | 非对齐启发式（可选，默认 `nullptr`） |
| `aligned_space` | `const Transform *` | 对齐空间变换（可选，默认 `nullptr`） |

**执行逻辑:**
- 当元素数量 < `BVH_SORT_THRESHOLD`（4096）时：直接使用 `sort()` 单线程排序
- 当元素数量 >= 4096 时：使用 `TaskPool` 进行多线程快速排序

### `bvh_reference_sort_threaded(task_pool, data, job_start, job_end, compare)`
多线程快速排序的内部实现（静态函数）。

**算法步骤:**
1. 若当前子数组大小 < 4096，退化为标准排序
2. 使用中位数三点法选取枢轴元素（首、中、末三个元素的中位数）
3. 执行分区操作，将数据分为左半部分和右半部分
4. 右半部分的排序任务推入 `TaskPool` 并行执行
5. 当前线程继续处理左半部分（迭代而非递归）
6. 特殊优化：如果左半部分为空，当前线程直接处理右半部分，避免不必要的任务调度

## 依赖关系

- **内部头文件**（sort.cpp）:
  - `bvh/sort.h` - 本文件的头文件
  - `bvh/params.h` - `BVHReference` 类定义
  - `bvh/unaligned.h` - `BVHUnaligned` 非对齐包围盒计算
  - `util/algorithm.h` - `sort()` 和 `swap()` 算法
  - `util/task.h` - `TaskPool` 并行任务管理
- **被引用**: `bvh/split.cpp`（对象分割和空间分割(Spatial Split)排序）

## 实现细节 / 关键算法

### 并行快速排序策略
排序采用混合并行策略：
- 大数组（>= 4096 元素）使用并行快速排序
- 小数组（< 4096 元素）使用 `std::sort` 避免线程调度开销
- 阈值 `BVH_SORT_THRESHOLD = 4096` 经过经验调优，避免小数组上频繁的线程休眠

### 中位数三点法（Median-of-Three）
选取数组首、中、末三个元素进行排序，取中间值作为枢轴。这种策略避免了在已排序或接近有序的数据上出现快速排序的最差性能（O(n^2)）。

### 任务分配优化
分区后不对称地创建新任务：
- 当前线程始终继续处理左半部分（通过循环迭代，非递归）
- 仅为右半部分创建新的任务
- 如果左半部分为空，当前线程直接接管右半部分的处理
- 这种策略减少了 `TaskScheduler` 的延迟开销，因为在同一线程上继续工作比创建新任务更高效

### 非对齐空间支持
当提供 `aligned_space` 变换矩阵时，排序比较基于变换后的包围盒坐标。这用于曲线等需要非轴对齐包围盒的图元类型，以获得更紧致的包围盒。

## 关联文件

- `bvh/params.h` - `BVHReference` 图元引用定义
- `bvh/unaligned.h` - 非对齐包围盒计算工具
- `bvh/split.h` / `bvh/split.cpp` - 分割策略，调用排序函数
- `bvh/build.h` / `bvh/build.cpp` - BVH 构建器，间接依赖排序
- `util/task.h` - `TaskPool` 并行任务管理
