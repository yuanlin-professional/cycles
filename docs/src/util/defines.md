# defines.h - 全局编译器与平台兼容宏定义

## 概述

`defines.h` 是 Cycles 渲染器最基础的宏定义头文件，提供 CPU/GPU 共享代码所需的编译器限定符（内联、对齐、地址空间）、平台检测（32/64 位）、分支预测提示、断言宏以及其他通用工具宏。几乎所有 Cycles 源文件通过 `util/types_base.h` 间接依赖本文件。

## 类与结构体

无类或结构体定义。

## 核心函数/宏定义

### 平台检测

| 宏 | 说明 |
|----|------|
| `__KERNEL_64_BIT__` | 在 64 位平台（x86-64、ARM64、IA64、PPC64）上定义 |

### CPU 端设备限定符（非 GPU 内核）

| 宏 | 展开 | 说明 |
|----|------|------|
| `ccl_device` | `static inline` | 设备函数（内联决策由编译器决定） |
| `ccl_device_extern` | `extern "C"` | 外部 C 链接设备函数 |
| `ccl_device_noinline` | `static inline` | 标记为不内联（实际仍 inline，由编译器决策） |
| `ccl_device_noinline_cpu` | `ccl_device_noinline` | CPU 专用不内联 |
| `ccl_device_inline` | `static __forceinline` (MSVC) / `static inline __attribute__((always_inline))` (GCC) | 强制内联 |
| `ccl_device_forceinline` | 同上 | 强制内联 |
| `ccl_device_inline_method` | `__forceinline` / `__attribute__((always_inline))` | 类方法强制内联 |
| `ccl_device_template_spec` | `template<> __forceinline` | 模板特化强制内联 |

### 内存对齐

| 宏 | 说明 |
|----|------|
| `ccl_align(...)` | 强制对齐：MSVC 用 `__declspec(align)`，GCC 用 `__attribute__((aligned))` |
| `ccl_try_align(...)` | 尝试对齐（32 位 MSVC 因函数参数限制无法对齐时为空） |

### GPU 地址空间占位符（CPU 端为空）

| 宏 | 说明 |
|----|------|
| `ccl_global` | 全局内存（GPU 端有实际语义） |
| `ccl_constant` | `const` |
| `ccl_private` | 私有内存 |
| `ccl_ray_data` | 光线数据内存 |
| `ccl_inline_constant` | `inline constexpr` |
| `ccl_static_constexpr` | `static constexpr` |
| `ccl_restrict` | `__restrict` |
| `ccl_optional_struct_init` | 空（GPU 端可能需要初始化） |
| `ccl_attr_maybe_unused` | `[[maybe_unused]]` |

### 编译器工具宏

| 宏 | 说明 |
|----|------|
| `LIKELY(x)` | 分支预测提示（GCC `__builtin_expect`） |
| `UNLIKELY(x)` | 分支预测提示（不太可能的路径） |
| `util_assert(statement)` | CPU 端为 `assert`，GPU 端为空操作 |
| `ATTR_FALLTHROUGH` | 抑制 `switch` 穿透警告（GCC 7+ 使用 `__attribute__((fallthrough))`） |
| `CONCAT(a, ...)` | 令牌拼接宏（通过辅助层确保宏参数展开） |
| `ccl_always_inline` | 强制内联属性 |
| `ccl_never_inline` | 禁止内联属性 |
| `ccl_may_alias` | 别名属性（GCC `__may_alias__`） |
| `ccl_ignore_integer_overflow` | 地址消毒器中忽略有符号整数溢出 |
| `__forceinline` | GCC/Clang 上的 `inline __attribute__((always_inline))` 兼容定义 |

### Metal 日志

| 宏 | 条件 | 说明 |
|----|------|------|
| `__METAL_PRINTF__` | Metal >= 3.2 | 启用 Metal 着色器日志 |
| `printf(...)` | `__METAL_PRINTF__` 下 | 重定向到 `metal::os_log_default.log_debug` |

## 依赖关系

- **内部头文件**: 无
- **外部依赖**: `<cassert>`（CPU 端）
- **被引用**: `util/types_base.h`, `util/texture.h`, `util/simd.h`, `util/stack_allocator.h`, `util/projection_inverse.h`, `util/math_int2.h`, `util/math_base.h`, `util/log.h`, `util/hash.h`, `kernel/util/nanovdb.h`, `kernel/tables.h`, `kernel/osl/types.h`, `kernel/sample/mis.h`

## 实现细节

1. **clang-format 禁用**：文件首行包含 `/* clang-format off */`，因为 `#define __forceinline` 会触发某些 clang-format 版本的格式化错误。

2. **32 位 MSVC SSE 对齐问题**：32 位 MSVC 不支持 SSE 对齐的函数参数（错误 C2719），因此 `ccl_try_align` 在此环境下定义为空，并取消 `__KERNEL_WITH_SSE_ALIGN__`。

3. **GPU/CPU 统一接口**：通过 `ccl_device`、`ccl_global` 等宏，同一份内核代码可以在 CPU 和 GPU 上编译。GPU 后端（CUDA/Metal/oneAPI）各自定义这些宏的实际语义。

4. **地址消毒器兼容**：`ccl_ignore_integer_overflow` 用于标记故意使用整数溢出的函数（如哈希函数），避免 AddressSanitizer 的误报。

## 关联文件

- `util/types_base.h` - 基础类型定义，包含本文件
- `util/simd.h` - SIMD 工具，依赖本文件的 `__forceinline` 等定义
- `util/optimization.h` - 指令集级别宏
