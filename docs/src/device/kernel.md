# kernel.h / kernel.cpp - 设备内核枚举与分类工具

## 概述

本文件定义了 Cycles 渲染器中所有设备内核（DeviceKernel）的分类查询函数和字符串转换工具。它提供了判断内核是否包含着色操作或相交测试的函数，以及将内核枚举值转换为人类可读字符串的功能。`DeviceKernelMask` 位集合类型用于批量追踪已入队的内核组合。该文件是波前路径追踪（Wavefront Path Tracing）架构中内核调度的基础支撑模块。

## 类与结构体

### DeviceKernelMask
- **继承**: `std::bitset<DEVICE_KERNEL_NUM>`
- **功能**: 内核位掩码集合，用于追踪一组已入队的内核组合，支持性能统计中按内核组合分类计时。
- **关键成员**: 继承自 `std::bitset` 的全部位操作
- **关键方法**:
  - `operator<(const DeviceKernelMask &other)` — 自定义比较运算符，用于在 `std::map` 中作为键排序

## 核心函数

- `device_kernel_has_shading(DeviceKernel kernel)` — 判断指定内核是否包含着色操作。返回 `true` 的内核包括：
  - 积分器着色内核：背景着色、光源着色、表面着色、表面光线追踪着色、MNEE 着色、体积着色、体积光线行进、阴影着色、专用光源着色
  - 着色器求值内核：位移求值、背景求值、曲线阴影透明度、体积密度

- `device_kernel_has_intersection(DeviceKernel kernel)` — 判断指定内核是否包含光线相交测试。返回 `true` 的内核包括：
  - 积分器相交内核：最近相交、阴影相交、次表面散射相交、体积堆栈相交、专用光源相交
  - 表面光线追踪着色、MNEE 着色（也包含相交操作）

- `device_kernel_as_string(DeviceKernel kernel)` — 将 `DeviceKernel` 枚举转换为对应的字符串名称，覆盖全部内核类型：
  - 积分器内核（初始化、相交、着色、排序、压缩、重置等）
  - 着色器求值内核
  - 胶片转换内核（通过 `FILM_CONVERT_KERNEL_AS_STRING` 宏批量生成）
  - 自适应采样内核
  - 降噪引导预处理内核
  - 体积散射概率引导内核
  - Cryptomatte 后处理内核
  - 通用前缀和内核

- `device_kernel_mask_as_string(DeviceKernelMask mask)` — 将内核位掩码集合转换为空格分隔的内核名称字符串，用于调试日志输出。

- `operator<<(std::ostream &os, DeviceKernel kernel)` — 流输出运算符重载，方便日志打印。

## 依赖关系

- **内部头文件**: `kernel/types.h`（定义了 `DeviceKernel` 枚举和 `DEVICE_KERNEL_NUM`）、`util/string.h`、`util/log.h`
- **外部库**: `<bitset>`、`<iosfwd>`
- **条件编译**: `__KERNEL_ONEAPI__` 宏控制部分代码的编译——oneAPI 后端直接在设备侧编译此文件，跳过主机端的流输出和日志相关代码
- **被引用**:
  - `src/device/queue.h` — 命令队列使用内核枚举和掩码进行调度与统计
  - `src/device/queue.cpp` — 使用字符串转换函数生成调试日志

## 实现细节 / 关键算法

- **宏批量生成**: `FILM_CONVERT_KERNEL_AS_STRING` 宏用于批量生成胶片转换内核的字符串映射，每个变体同时生成标准版本和半精度 RGBA 版本（`_half_rgba` 后缀），减少重复代码。
- **oneAPI 双编译兼容**: 使用 `#ifndef __KERNEL_ONEAPI__` 条件编译使该文件既能在主机端编译（包含完整的日志和流操作），也能在 oneAPI 设备端编译（仅保留 `device_kernel_as_string` 核心函数）。
- **DeviceKernelMask 比较**: 自定义的 `operator<` 按位从低到高扫描，在第一个差异位处根据 `other` 是否设置该位来决定排序顺序，保证在 `std::map` 中的确定性排序。

## 关联文件

- `src/kernel/types.h` — 定义 `DeviceKernel` 枚举和 `DEVICE_KERNEL_NUM` 常量
- `src/device/queue.h` / `queue.cpp` — 使用内核枚举进行命令入队和性能统计
- `src/integrator/path_trace_work_gpu.cpp` — 波前路径追踪中调度各阶段内核
- `src/integrator/shader_eval.cpp` — 着色器求值中使用内核枚举
