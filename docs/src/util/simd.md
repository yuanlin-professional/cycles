# simd.h - SIMD 内联函数与位操作工具

## 概述

`simd.h` 提供 Cycles 渲染器使用的 SIMD（Single Instruction Multiple Data）基础设施，包括 SSE/AVX 内联函数头文件的统一包含、浮点控制模式设置（Flush-to-Zero / Denormals-are-Zero）、SSE 辅助数据类型、ARM Neon 的 shuffle 工具函数，以及跨平台的位扫描 (BSF/BSR/BTC) 内联函数。此文件源自 Intel Embree 项目。

## 类与结构体

### SSE 辅助类型（仅 `__KERNEL_SSE2__` 环境下）

| 类型 | 实例名 | 说明 |
|------|--------|------|
| `TrueTy` | `True` | 可隐式转换为 `bool(true)` |
| `FalseTy` | `False` | 可隐式转换为 `bool(false)` |
| `ZeroTy` | `zero` | 可转换为 `float(0)` 或 `int(0)` |
| `OneTy` | `one` | 可转换为 `float(1)` 或 `int(1)` |
| `NegInfTy` | `neg_inf` | 可转换为负无穷 float 或 int 最小值 |
| `PosInfTy` | `inf` / `pos_inf` | 可转换为正无穷 float 或 int 最大值 |
| `StepTy` | `step` | 步进标记类型 |

### 外部变量

| 变量 | 说明 |
|------|------|
| `extern const __m128 _mm_lookupmask_ps[16]` | 16 元素查找表掩码数组 |

## 核心函数/宏定义

### 浮点控制宏

| 宏 | 说明 |
|----|------|
| `SIMD_SET_FLUSH_TO_ZERO` | 启用 Flush-to-Zero 和 Denormals-are-Zero 模式，提高浮点性能（避免非规格化数惩罚） |

**平台实现：**
- x86-64：使用 `_MM_SET_FLUSH_ZERO_MODE` 和 `_MM_SET_DENORMALS_ZERO_MODE`
- AArch64（sse2neon >= 1.5.0）：使用 sse2neon 提供的兼容函数
- AArch64（旧版本）：直接操作 FPCR 寄存器
- AArch64（MSVC）：使用 `_ReadStatusReg` / `_WriteStatusReg`
- 其他平台：空操作

### ARM Neon Shuffle 函数

```cpp
template<class type, int i0, int i1, int i2, int i3>
type shuffle_neon(const type &a);

template<class type, int i0, int i1, int i2, int i3>
type shuffle_neon(const type &a, const type &b);
```

模板化的 Neon 向量 shuffle 实现，模拟 x86 `_mm_shuffle_ps` 行为：
- 单参数版本：从同一向量中按索引重排
- 双参数版本：从两个向量中各取元素（同源优化路径 + 双源 `vqtbl2q_s8` 路径）
- 所有元素相同时使用 `vdupq_laneq_s32` 快速路径

### 位操作内联函数

| 函数 | 说明 |
|------|------|
| `__bsf(v)` | Bit Scan Forward - 查找最低位 1 的位置 |
| `__bsr(v)` | Bit Scan Reverse - 查找最高位 1 的位置 |
| `__btc(v, i)` | Bit Test and Complement - 翻转指定位 |
| `bitscan(v)` | 位扫描（有 AVX2 时使用 `_tzcnt`，否则回退到 `__bsf`） |

**三套实现：**
1. **Windows MSVC**：使用 `_BitScanForward`/`_BitScanReverse`/`_bittestandcomplement` 内联函数
2. **x86 GCC/Clang**：使用内联汇编 `bsf`/`bsr`/`btc` 指令
3. **通用回退**：逐位循环扫描

每个函数均有 `uint32_t` 和 `uint64_t` 两个重载版本。

### SSE 内联函数包含

根据平台选择合适的头文件：
- MinGW64：使用 `util/windows.h`（避免重复声明冲突）
- MSVC（非 Neon）：使用 `<intrin.h>`
- GCC/Clang x86：使用 `<x86intrin.h>`
- ARM Neon：使用 `<sse2neon.h>`（需定义 `SSE2NEON_PRECISE_MINMAX`）

### AVX 兼容宏

```cpp
#define _mm256_cvtss_f32(a) (_mm_cvtss_f32(_mm256_castps256_ps128(a)))
```

针对旧版 GCC 缺少 `_mm256_cvtss_f32` 的兼容定义。

## 依赖关系

- **内部头文件**: `util/defines.h`
- **外部依赖**: `<cstdint>`, `<limits>`, `<intrin.h>`/`<x86intrin.h>`/`<sse2neon.h>`（平台相关）
- **被引用**: `util/types_base.h`, `util/half.h`, `kernel/device/metal/compat.h`

## 实现细节

1. **Flush-to-Zero 性能**：非规格化浮点数在 x86 上可导致严重性能惩罚（10-100 倍减速）。Embree 光线追踪库要求启用 FTZ/DAZ 模式，本文件在初始化时统一设置。

2. **Neon shuffle 设计**：x86 的 `_mm_shuffle_ps` 在 ARM 上无直接对应指令。本文件使用 `vqtbl1q_s8`（查表）实现通用 shuffle，并为所有元素相同的特殊情况提供 `vdupq_laneq_s32` 快速路径。双源 shuffle 使用 `vqtbl2q_s8`。

3. **AVX2 尾部零计数**：当 AVX2 可用时，`bitscan` 使用 `_tzcnt_u32`/`_tzcnt_u64`（BMI1 指令），比传统 `bsf` 更快且输入为 0 时行为确定。

4. **MSVC ARM64 兼容**：ARM64 上的 Neon shuffle 实现需注意 MSVC 将某些 Neon 内联函数实现为多层宏，不能在单行表达式中使用。

## 关联文件

- `util/defines.h` - 提供 `__KERNEL_64_BIT__`、`__forceinline` 等基础宏
- `util/optimization.h` - 定义 `__KERNEL_SSE42__`、`__KERNEL_NEON__` 等指令集级别宏
- `util/types_base.h` - 包含本文件以获取 SIMD 类型支持
