# CMakeLists.txt - Cycles 测试套件构建配置

## 概述

此 CMakeLists.txt 文件定义了 Cycles 渲染器测试套件的构建系统配置。它负责编译和链接所有单元测试源文件，并在启用 Google Test (GTest) 时生成可执行测试程序。

## 构建选项

### 编译器标志调整
- 当 `WITH_GTESTS` 为 `ON` 时，调用 `remove_strict_flags()` 移除严格编译警告标志，以避免外部项目（如 GTest）中无法修复的警告。

### AVX 指令集条件编译
- 在非 Apple 平台上，若编译器支持 AVX2 指令集（`CXX_HAS_AVX2`），则额外编译 `util_float8_avx2_test.cpp`，并为其设置 `CYCLES_AVX2_FLAGS` 编译标志。
- macOS 平台跳过 AVX 测试，因为 Rosetta 转译层运行此类测试存在问题。

## 包含目录

| 变量 | 路径 | 说明 |
|------|------|------|
| `INC` | `..` | 父目录，即 `src/` 目录，用于引用 Cycles 头文件 |

## 链接库

测试可执行文件链接以下 Cycles 内部库：

| 库名称 | 说明 |
|--------|------|
| `cycles_kernel` | 渲染核心 |
| `cycles_integrator` | 积分器模块 |
| `cycles_scene` | 场景管理模块 |
| `cycles_session` | 会话管理模块 |
| `cycles_bvh` | 包围盒层次结构（BVH）模块 |
| `cycles_graph` | 图形/节点图模块 |
| `cycles_subd` | 细分曲面模块 |
| `cycles_device` | 设备抽象层 |
| `cycles_util` | 通用工具库 |

此外，通过 `cycles_external_libraries_append(LIB)` 追加外部依赖库。

## 测试目标

当 `WITH_GTESTS` 为 `ON` 时，通过 `blender_add_test_suite_executable(cycles ...)` 创建名为 `cycles` 的测试套件可执行文件，包含以下测试源文件：

### 核心测试文件
- `integrator_adaptive_sampling_test.cpp` - 自适应采样测试
- `integrator_render_scheduler_test.cpp` - 渲染调度器测试
- `integrator_tile_test.cpp` - 图块计算测试
- `kernel_camera_projection_test.cpp` - 相机投影测试
- `render_graph_finalize_test.cpp` - 着色器图最终化测试

### 工具库测试文件
- `util_aligned_malloc_test.cpp` - 对齐内存分配测试
- `util_boundbox_test.cpp` - 包围盒测试
- `util_ies_test.cpp` - IES 光域文件测试
- `util_math_test.cpp` - 数学函数测试
- `util_math_fast_test.cpp` - 快速数学函数测试
- `util_math_float3_test.cpp` - float3 数学运算测试
- `util_math_float4_test.cpp` - float4 数学运算测试
- `util_rgbe_test.cpp` - RGBE 编码测试
- `util_md5_test.cpp` - MD5 哈希测试
- `util_path_test.cpp` - 路径工具测试
- `util_string_test.cpp` - 字符串工具测试
- `util_task_test.cpp` - 任务调度测试
- `util_time_test.cpp` - 时间工具测试
- `util_transform_test.cpp` - 变换矩阵测试

### 条件编译测试文件（非 Apple 平台 + AVX2 支持）
- `util_float8_avx2_test.cpp` - float8 AVX2 指令集测试

## 关联文件
- `src/test/` 目录下的所有 `.cpp` 测试源文件
- Cycles 各模块的库文件
