# aligned_malloc.h / aligned_malloc.cpp - 对齐内存分配工具

## 概述
本文件提供跨平台的对齐内存分配与释放功能，满足 SSE/AVX 等 SIMD 指令集和 NanoVDB 等库的内存对齐要求。支持 Windows（`_aligned_malloc`）、macOS/BSD（`posix_memalign`）和 Linux（`memalign`）三种平台后端，并可选使用 Blender 的 `MEM_guardedalloc` 作为分配后端。所有分配和释放操作都会记录到 `GuardedAllocator` 的内存统计系统。

## 类与结构体
无自定义类。

### 预定义对齐常量
| 宏 | 值 | 说明 |
|------|------|------|
| `MIN_ALIGNMENT_CPU_DATA_TYPES` | 16 | CPU 原生数据类型最小对齐（SSE/AVX） |
| `MIN_ALIGNMENT_DEVICE_MEMORY` | 32 | 设备内存最小对齐（NanoVDB 要求） |

## 核心函数

### 基础分配/释放
| 函数 | 说明 |
|------|------|
| `void *util_aligned_malloc(size_t size, int alignment)` | 分配指定对齐的内存块 |
| `void util_aligned_free(void *ptr, size_t size)` | 释放对齐内存块 |

#### `util_aligned_malloc` 平台实现
| 条件 | 实现 |
|------|------|
| `WITH_BLENDER_GUARDEDALLOC` | `MEM_mallocN_aligned(size, alignment, "Cycles Aligned Alloc")` |
| Windows | `_aligned_malloc(size, alignment)` |
| macOS/FreeBSD/NetBSD/OpenBSD | `posix_memalign(&mem, alignment, size)` |
| Linux | `memalign(alignment, size)` |

#### `util_aligned_free` 平台实现
| 条件 | 实现 |
|------|------|
| `WITH_BLENDER_GUARDEDALLOC` | `MEM_freeN(ptr)` |
| Windows | `_aligned_free(ptr)` |
| 其他 | `free(ptr)` |

### 对齐 new/delete 模板
| 函数 | 说明 |
|------|------|
| `T *util_aligned_new<T>(Args... args)` | 使用 `alignof(T)` 对齐分配并构造对象（placement new） |
| `void util_aligned_delete<T>(T *t)` | 调用析构函数并释放对齐内存 |

### 内存统计集成
- 分配成功后调用 `util_guarded_mem_alloc(size)` 记录
- 释放前调用 `util_guarded_mem_free(size)` 记录

## 依赖关系
- **内部头文件**: `util/guarded_allocator.h`
- **外部依赖**: `MEM_guardedalloc.h`（Blender 嵌入时）, `<malloc.h>`（Linux/Windows）
- **被引用**: `device/` 下的设备内存管理, `scene/` 下的场景数据分配

## 关联文件
- `util/guarded_allocator.h` - `util_guarded_mem_alloc/free` 统计接口
- `util/stats.h` - 内存统计底层实现
