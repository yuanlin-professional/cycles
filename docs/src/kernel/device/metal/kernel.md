# kernel.metal - Metal 内核入口与 MetalRT 求交处理函数

## 概述

本文件是 Cycles 渲染器 Metal 后端的主内核源文件，编译为 Metal Shading Language (MSL) 着色器库。它组装所有必要的头文件以构建完整的 Metal 内核，并在启用 MetalRT 时定义大量求交处理函数（Intersection Functions），用于在硬件光线追踪过程中执行自定义的几何体过滤和命中处理逻辑。

文件分为两个主要部分：头文件包含链（组装内核）和 MetalRT 求交处理函数（自定义命中逻辑）。

## 核心函数/宏定义

### 头文件包含链

1. `kernel/device/metal/compat.h` — Metal 兼容层（必须最先包含）
2. `kernel/device/metal/globals.h` — 全局数据结构
3. `kernel/device/metal/function_constants.h` — 函数常量（必须在 kernel.h 之前）
4. `kernel/device/gpu/kernel.h` — GPU 通用内核代码（包含所有渲染内核实现）

### MetalRT 返回类型结构体

| 结构体 | 说明 |
|--------|------|
| `BoundingBoxIntersectionResult` | 包围盒求交返回值：`accept`、`continue_search`、`distance` |
| `PrimitiveIntersectionResult` | 图元求交返回值：`accept`、`continue_search` |

### 求交类型枚举

```cpp
enum { METALRT_HIT_TRIANGLE, METALRT_HIT_CURVE, METALRT_HIT_BOUNDING_BOX };
```

### 三角形求交处理函数

| 函数 | 标签 | 说明 |
|------|------|------|
| `__intersection__tri` | `triangle, METALRT_TAGS` | 默认三角形可见性测试 |
| `__intersection__tri_shadow` | `triangle, METALRT_TAGS` | 阴影射线三角形测试 |
| `__intersection__tri_shadow_all` | `triangle, METALRT_TAGS` | 全阴影记录三角形测试 |
| `__intersection__volume_tri` | `triangle, METALRT_TAGS` | 体积三角形测试（仅接受含体积属性的对象） |
| `__intersection__local_tri` | `triangle` | 局部求交三角形测试（BLAS 级别） |
| `__intersection__local_tri_single_hit` | `triangle` | 局部单命中三角形测试 |
| `__intersection__local_tri_mblur` | `triangle, METALRT_TAGS` | 局部运动模糊三角形测试 |
| `__intersection__local_tri_single_hit_mblur` | `triangle, METALRT_TAGS` | 局部单命中运动模糊测试 |

### 曲线求交处理函数

| 函数 | 说明 |
|------|------|
| `__intersection__curve` | 默认曲线可见性测试，含端盖过滤和带状曲线接受检查 |
| `__intersection__curve_shadow` | 阴影射线曲线测试 |
| `__intersection__curve_shadow_all` | 全阴影记录曲线测试 |

### 点云求交处理函数（`__POINTCLOUD__`）

| 函数 | 说明 |
|------|------|
| `__intersection__point` | 点图元可见性测试（包围盒型） |
| `__intersection__point_shadow` | 阴影射线点测试 |
| `__intersection__point_shadow_all` | 全阴影记录点测试 |

### 核心模板函数

| 函数 | 说明 |
|------|------|
| `metalrt_visibility_test<TReturn, intersection_type>()` | 通用可见性测试模板，检查对象可见性、曲线端盖、带状接受和自相交 |
| `metalrt_visibility_test_shadow<TReturn, intersection_type>()` | 阴影可见性测试模板，额外处理阴影链接和自阴影跳过 |
| `metalrt_shadow_all_hit<intersection_type>()` | 全阴影命中处理模板，记录透明阴影命中并管理吞吐量 |
| `metalrt_local_hit<TReturn, intersection_type>()` | 局部命中处理模板，支持随机采样的多命中记录 |

### 辅助函数

| 函数 | 说明 |
|------|------|
| `metalrt_curve_skip_end_cap()` | 判断是否应跳过曲线端盖（u=0或u=1时，非厚线性曲线） |
| `metalrt_intersection_point_shadow_all()` | 点云全阴影求交辅助函数 |

## 依赖关系

- **内部头文件**:
  - `kernel/device/metal/compat.h` — Metal 兼容层
  - `kernel/device/metal/globals.h` — 全局数据结构
  - `kernel/device/metal/function_constants.h` — 函数常量
  - `kernel/device/gpu/kernel.h` — GPU 通用内核代码
- **被引用**:
  - `device/metal/device_impl.mm` — 主机端将本文件路径嵌入编译源码字符串：`#include "kernel/device/metal/kernel.metal"`

## 实现细节 / 关键算法

### MetalRT 求交函数表机制

Metal 的硬件光线追踪使用求交函数表（Intersection Function Table）来注册自定义的命中处理逻辑。每种求交场景对应不同的函数表：

| 函数表（在 MetalAncillaries 中） | 对应的求交处理函数 |
|----------------------------------|-------------------|
| `ift_default` | `__intersection__tri`, `__intersection__curve`, `__intersection__point` |
| `ift_shadow` | `__intersection__tri_shadow`, `__intersection__curve_shadow`, `__intersection__point_shadow` |
| `ift_shadow_all` | `__intersection__tri_shadow_all`, `__intersection__curve_shadow_all`, `__intersection__point_shadow_all` |
| `ift_volume` | `__intersection__volume_tri` |
| `ift_local` | `__intersection__local_tri` |
| `ift_local_single_hit` | `__intersection__local_tri_single_hit` |
| `ift_local_mblur` | `__intersection__local_tri_mblur` |
| `ift_local_single_hit_mblur` | `__intersection__local_tri_single_hit_mblur` |

### 全阴影记录算法

`metalrt_shadow_all_hit` 模板实现了透明阴影的完整记录逻辑：
1. 检查对象可见性
2. 跳过自相交和阴影链接排除
3. 检查是否为不透明表面（无透明阴影标志时直接终止）
4. 曲线图元使用烘焙的阴影透明度衰减吞吐量
5. 管理有限的命中记录缓冲区，超出容量时替换最远的命中
6. 透明命中次数超过上限时终止

### 局部求交的蓄水池采样

在多命中模式下，当命中数超过 `max_hits` 时，使用 LCG 随机数生成器进行蓄水池采样（Reservoir Sampling），确保每个命中被选中的概率相等。

## 关联文件

- `kernel/device/metal/bvh.h` — MetalRT 场景求交函数（`scene_intersect` 等）
- `kernel/device/metal/compat.h` — MetalRT 类型定义和兼容层
- `kernel/device/metal/context_begin.h` / `context_end.h` — MetalKernelContext 类
- `kernel/device/gpu/kernel.h` — GPU 通用内核实现
- `device/metal/device_impl.mm` — 主机端编译和管线管理
