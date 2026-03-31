# types.h - BVH 遍历类型定义与宏常量

## 概述

`types.h` 是 BVH 子系统的基础定义文件，提供了所有 BVH 遍历模板文件所需的常量、宏和预处理器工具。该文件定义了遍历栈大小、哨兵值、特性位掩码以及用于生成多变体函数名的宏拼接机制。它是 BVH 宏模板编译系统的核心组成部分，被 `bvh.h` 在所有遍历模板之前引入。

## 类与结构体

本文件不定义类或结构体。

## 枚举与常量

### 交叉函数修饰符

| 宏 | GPU 定义 | CPU 定义 | 说明 |
|---|---|---|---|
| `ccl_device_intersect` | `ccl_device_forceinline` | `ccl_device_inline` | GPU 上强制内联交叉函数以提升性能 |

### 遍历栈

| 常量 | 值 | 说明 |
|---|---|---|
| `ENTRYPOINT_SENTINEL` | `0x76543210` | 遍历栈底部哨兵值，标志遍历结束 |
| `BVH_STACK_SIZE` | `192` | 遍历栈最大深度（64 物体 BVH + 64 网格 BVH + 64 物体节点分裂） |

### 特性位掩码

| 常量 | 值 | 说明 |
|---|---|---|
| `BVH_MOTION` | `1` | 运动模糊支持标志 |
| `BVH_HAIR` | `2` | 毛发曲线支持标志（启用非轴对齐节点交叉） |
| `BVH_POINTCLOUD` | `4` | 点云支持标志 |

### 宏模板工具

| 宏 | 说明 |
|---|---|
| `BVH_NAME_JOIN(x, y)` | 拼接两个标记为 `x_y`（使用 `##` 运算符） |
| `BVH_NAME_EVAL(x, y)` | 间接拼接（确保宏参数先展开再拼接） |
| `BVH_FUNCTION_FULL_NAME(prefix)` | 将前缀与当前 `BVH_FUNCTION_NAME` 拼接为完整函数名 |
| `BVH_FEATURE(f)` | 检查当前编译变体是否启用了特性 `f`（测试 `BVH_FUNCTION_FEATURES & f`） |

## 核心函数

本文件不定义函数，仅提供宏定义。

## 依赖关系

### 内部头文件
无直接依赖（使用 `#pragma once` 保护）。

### 被引用
- `src/kernel/bvh/bvh.h` - 直接 `#include "kernel/bvh/types.h"`
- `src/kernel/device/metal/bvh.h` - Metal 设备 BVH 实现
- `src/kernel/device/optix/bvh.h` - OptiX 设备 BVH 实现
- `src/kernel/device/cpu/bvh.h` - CPU/Embree 设备 BVH 实现

## 实现细节 / 关键算法

### 宏模板系统工作原理

BVH 遍历的多变体编译依赖以下协作机制：

1. **调用方**（`bvh.h`）在每次 `#include` 遍历模板前定义：
   - `BVH_FUNCTION_NAME`: 如 `bvh_intersect_hair_motion`
   - `BVH_FUNCTION_FEATURES`: 如 `BVH_HAIR | BVH_MOTION | BVH_POINTCLOUD`

2. **模板文件**（如 `traversal.h`）使用本文件定义的宏：
   - `BVH_FUNCTION_FULL_NAME(BVH)` 展开为 `BVH_bvh_intersect_hair_motion`（内部实现函数名）
   - `BVH_FEATURE(BVH_HAIR)` 展开为 `((BVH_HAIR | BVH_MOTION | BVH_POINTCLOUD) & BVH_HAIR) != 0`，即 `true`

3. **模板文件末尾** `#undef BVH_FUNCTION_NAME` 和 `#undef BVH_FUNCTION_FEATURES`，为下次包含做准备

### 栈大小设计

`BVH_STACK_SIZE = 192` 的设计考虑了三层嵌套：
- 64 级：顶层场景 BVH（物体级）
- 64 级：物体内部网格 BVH
- 64 级：因空间分裂（spatial split）可能产生的额外节点

### ENTRYPOINT_SENTINEL 设计

哨兵值 `0x76543210` 被选为一个不太可能出现在正常节点地址中的值。它既不是有效的正数节点地址，也不是有效的负数叶节点地址，因此可以安全地用作栈底标记和实例 BVH 边界标记。

## 关联文件

| 文件 | 关系 |
|---|---|
| `kernel/bvh/bvh.h` | 引入本文件并使用其宏定义 |
| `kernel/bvh/traversal.h` | 使用 `BVH_FUNCTION_FULL_NAME`、`BVH_FEATURE` 等宏 |
| `kernel/bvh/local.h` | 使用相同宏体系 |
| `kernel/bvh/shadow_all.h` | 使用相同宏体系 |
| `kernel/bvh/volume.h` | 使用相同宏体系 |
| `kernel/bvh/volume_all.h` | 使用相同宏体系 |
