# test - 单元测试

## 概述

`src/test/` 模块包含 Cycles 渲染引擎的单元测试套件，基于 Google Test（GTest）框架编写。测试覆盖积分器（自适应采样、渲染调度、分块）、内核（相机投影）、场景（着色器图优化）以及大量工具库函数（数学运算、路径处理、字符串操作、SIMD 向量运算等）。

所有测试通过 CMake 的 `blender_add_test_suite_executable` 宏统一构建为 `cycles` 测试可执行文件，需启用 `WITH_GTESTS` 编译选项。

## 测试用例列表

| 测试文件 | 测试内容 | 验证范围 |
|----------|----------|----------|
| `integrator_adaptive_sampling_test.cpp` | 自适应采样逻辑 | 验证 `AdaptiveSampling` 类的 `align_samples()`（采样数对齐到过滤点）、`need_filter()`（判断是否需要过滤的采样序列）和 `schedule_samples()`（调度采样数后必须处于过滤点） |
| `integrator_render_scheduler_test.cpp` | 渲染调度器分辨率计算 | 验证 `calculate_resolution_divider_for_resolution()`（从目标分辨率计算分辨率除数）和 `calculate_resolution_for_divider()`（从除数反算分辨率） |
| `integrator_tile_test.cpp` | 分块大小计算 | 验证 `tile_calculate_best_size()` 在不同条件下（CPU/GPU、不同图像分辨率、采样数、路径状态数）的最优分块尺寸计算，包括常规和极端情况 |
| `kernel_camera_projection_test.cpp` | 相机投影变换 | 验证多种全景相机投影模型的正向/反向变换一致性（往返测试）：鱼眼多项式镜头（`fisheye_lens_polynomial`）、球面（`spherical`）、等距矩形（`equirectangular`）、等距鱼眼（`fisheye_equidistant`）、等固体角鱼眼（`fisheye_equisolid`）、镜像球（`mirrorball`）、等角立方体面（`equiangular_cubemap_face`），以及对称性和手工参考值验证 |
| `render_graph_finalize_test.cpp` | 着色器图优化 | 验证 `ShaderGraph::finalize()` 的节点优化和常量折叠，覆盖：单值图像纹理去除、数学节点常量折叠（加/减/乘/除/幂/对数等）、Mix 节点优化（零因子/一因子/相同输入）、向量数学运算折叠、无效连接处理、代理节点展开等大量场景 |
| `util_aligned_malloc_test.cpp` | 对齐内存分配 | 验证 `util_aligned_malloc()` 返回的指针满足 16 字节和 32 字节对齐要求 |
| `util_boundbox_test.cpp` | 包围盒变换 | 验证 `BoundBox::transformed()` 对包围盒施加变换矩阵后的正确性，以及无效包围盒变换后仍然无效 |
| `util_float8_test.h` | vfloat8 向量运算（共享测试头） | 定义 8 分量浮点向量的测试用例：算术运算（加/减/乘/除，向量-向量和向量-标量）、构造函数、平方根、最小/最大值、shuffle 操作，含 CPU 指令集能力检测 |
| `util_float8_avx2_test.cpp` | AVX2 指令集下的 vfloat8 | 在定义 `__KERNEL_AVX2__` 宏后包含 `util_float8_test.h`，验证 AVX2 路径下的 vfloat8 运算正确性 |
| `util_float8_avx_test.cpp` | AVX 指令集下的 vfloat8 | 在定义 `__KERNEL_AVX__` 宏后包含 `util_float8_test.h`，验证 AVX 路径下的 vfloat8 运算正确性 |
| `util_float8_sse2_test.cpp` | SSE2 指令集下的 vfloat8 | 在定义 `__KERNEL_SSE2__` 宏后包含 `util_float8_test.h`，验证 SSE2 回退路径下的 vfloat8 运算正确性 |
| `util_ies_test.cpp` | IES 光域网文件解析 | 验证 `IESFile::load()` 对无效输入数据的错误处理（返回 false） |
| `util_math_test.cpp` | 基础数学工具函数 | 验证 `next_power_of_two()`（下一个 2 的幂）、`prev_power_of_two()`（上一个 2 的幂）和 `reverse_integer_bits()`（整数位反转）的正确性 |
| `util_math_fast_test.cpp` | 快速数学近似函数 | 验证 `fast_sinf()` 和 `fast_cosf()` 在 -2PI..2PI 范围内与标准库函数的精度偏差，以及特殊值（PI/2）和大数值的范围归约精度 |
| `util_math_float3_test.cpp` | float3 向量数学运算 | 验证 `fmod(float3, float)` 在多种数值范围下的正确性，包括正/负值、零值、大数值边界情况 |
| `util_math_float4_test.cpp` | float4 向量数学运算 | 验证 `fmod(float4, float)` 在多种数值范围下的正确性，包括正/负值、零值、大数值边界情况 |
| `util_md5_test.cpp` | MD5 哈希计算 | 验证 `util_md5_string()` 对已知字符串 "Hello, World!" 的 MD5 哈希值计算正确性 |
| `util_path_test.cpp` | 文件路径操作 | 验证 `path_filename()`（提取文件名）、`path_dirname()`（提取目录名）、`path_join()`（路径拼接）、`path_escape()`（空格转义）、`path_is_relative()`（相对路径判断）在 Unix 和 Windows 平台下的正确行为 |
| `util_rgbe_test.cpp` | RGBE 颜色编码 | 验证 `rgb_to_rgbe()` 和 `rgbe_to_rgb()` 的往返转换精度，覆盖正常值、零值、极小值、负值、极大值（FLT_MAX）和无穷大的处理 |
| `util_string_test.cpp` | 字符串工具函数 | 验证 `string_printf()`（格式化）、`string_iequals()`（大小写无关比较）、`string_split()`（空白分割）、`string_replace()`（子串替换）、`string_endswith()`/`string_startswith()`（前后缀判断）、`string_strip()`（去空白）、`string_remove_trademark()`（去除 (TM)/(R) 标记） |
| `util_task_test.cpp` | 任务调度系统 | 验证 `TaskScheduler` 和 `TaskPool` 的基本功能：初始化/退出、提交 100 个任务并等待完成、任务计数验证，以及重复初始化/退出的稳定性（1000 次循环） |
| `util_time_test.cpp` | 时间格式转换 | 验证 `time_human_readable_to_seconds()` 和 `time_human_readable_from_seconds()` 的双向转换：空串、纯小数、秒、分:秒、时:分:秒格式 |
| `util_transform_test.cpp` | 变换矩阵分解 | 验证 `transform_motion_decompose()` 对退化矩阵（全零缩放）的处理：单个退化矩阵的安全分解、从前一帧/后一帧拷贝到退化帧的回退逻辑 |

## 依赖关系

### 上游依赖（本模块依赖）

- Google Test（GTest）- 单元测试框架
- `cycles_kernel` - 渲染内核（相机投影测试）
- `cycles_integrator` - 积分器（自适应采样、调度器、分块测试）
- `cycles_scene` - 场景模块（着色器图优化测试）
- `cycles_session` - 渲染会话
- `cycles_bvh` - BVH 加速结构
- `cycles_graph` - 节点图系统
- `cycles_subd` - 细分曲面
- `cycles_device` - 设备层
- `cycles_util` - 工具库（大部分工具测试的直接测试目标）

### 下游依赖（依赖本模块）

- 无 - 测试模块为终端构建目标，不被其他模块依赖

## 参见

- `src/integrator/` - 被测试的积分器模块（自适应采样、渲染调度、分块）
- `src/kernel/camera/` - 被测试的相机投影内核
- `src/scene/shader_graph.h` - 被测试的着色器图优化
- `src/util/` - 被测试的工具库（数学、路径、字符串、任务、时间、变换等）
