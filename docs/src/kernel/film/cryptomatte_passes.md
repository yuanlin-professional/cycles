# cryptomatte_passes.h - Cryptomatte 遮罩通道的槽位写入与排序

## 概述

本文件实现了 Cryptomatte 渲染通道的底层槽位（slot）管理逻辑。Cryptomatte 是一种行业标准技术，用于在渲染时为每个像素记录覆盖该像素的物体、材质或资产的 ID 及其对应权重，以便后期合成时精确创建遮罩。

文件定义了 Cryptomatte 缓冲区元素结构体，并提供了槽位写入（支持原子操作）和按权重排序的功能。

## 类与结构体

### CryptoPassBufferElement
```c
struct CryptoPassBufferElement {
  float x;  // Cryptomatte ID（哈希值）
  float y;  // 该 ID 对应的累积权重
};
```
- **功能**: 表示 Cryptomatte 渲染缓冲区中的一个元素对。语义上等同于 `float2`，但使用结构体而非向量类型，原因是 ID 通道在渲染缓冲区中的偏移可能不满足编译器对对齐的要求。
- `x` 存储浮点编码的 Cryptomatte ID
- `y` 存储该 ID 在当前像素上的累积权重

## 枚举与常量

- `ID_NONE` — 表示空槽位的哨兵值，用于判断某个槽位是否已被占用

## 核心函数

### film_write_cryptomatte_slots()
- **签名**: `ccl_device_inline void film_write_cryptomatte_slots(ccl_global float *buffer, const int num_slots, const float id, const float weight)`
- **功能**: 将指定的 Cryptomatte ID 和权重写入缓冲区槽位。逻辑如下：
  1. 遍历所有槽位，查找匹配的 ID 或空槽位
  2. 如果找到空槽位：在 GPU 模式下使用 `atomic_compare_and_swap_float` 原子操作争夺该槽位；在 CPU 模式下直接写入
  3. 如果找到匹配 ID 的槽位或已到达最后一个槽位：累加权重
  4. 权重为 0 时直接返回

### film_sort_cryptomatte_slots()
- **签名**: `ccl_device_inline void film_sort_cryptomatte_slots(ccl_global float *buffer, const int num_slots)`
- **功能**: 对 Cryptomatte 槽位按权重进行降序排序。使用插入排序算法，因为槽位数量极少（通常 4-8 个），插入排序在此场景下效率足够。排序确保权重最大的 ID 排在最前面。

### film_cryptomatte_post()
- **签名**: `ccl_device_inline void film_cryptomatte_post(KernelGlobals kg, ccl_global float *render_buffer, const int pixel_index)`
- **功能**: Cryptomatte 后处理函数。在渲染完成后对指定像素的 Cryptomatte 缓冲区执行排序操作。槽位数量由 `kernel_data.film.cryptomatte_depth * 2` 决定。

## 依赖关系

- **内部头文件**:
  - `kernel/globals.h` — 提供 `KernelGlobals` 全局内核数据访问

- **被引用**:
  - `src/kernel/film/data_passes.h` — 数据通道写入时调用 `film_write_cryptomatte_slots()` 写入物体/材质/资产 Cryptomatte 数据
  - `src/kernel/device/cpu/kernel_arch_impl.h` — CPU 内核入口调用 `film_cryptomatte_post()` 进行后排序

## 实现细节 / 关键算法

### 原子写入机制

在 GPU 模式下（`__ATOMIC_PASS_WRITE__` 已定义），Cryptomatte 槽位写入使用无锁并发策略：

1. **槽位争夺**: 使用 `atomic_compare_and_swap_float` 尝试将空槽位的 ID 从 `ID_NONE` 替换为目标 ID。如果另一个线程先占了该槽位且写入了不同的 ID，则继续检查下一个槽位。
2. **权重累加**: 使用 `atomic_add_and_fetch_float` 原子地累加权重值。

在 CPU 模式下，由于像素级别的串行处理，直接使用普通赋值和加法操作。

### 插入排序

排序使用标准插入排序，遇到 `ID_NONE` 时提前终止。由于 Cryptomatte 深度通常很小（默认 2 层，即 4 个槽位），排序开销可忽略。

## 关联文件

- `src/kernel/film/data_passes.h` — 调用本文件的槽位写入函数完成 Cryptomatte 数据的实际记录
- `src/kernel/film/write.h` — 渲染通道写入基础设施
- `src/kernel/device/cpu/kernel_arch_impl.h` — CPU 设备后处理入口
- `src/kernel/device/gpu/kernel.h` — GPU 设备内核（间接通过 data_passes 使用）
