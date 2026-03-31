# kernel/light - 光源采样与求交

## 概述

`kernel/light` 模块实现了 Cycles 路径追踪渲染器中所有光源类型的采样、求交和 PDF 评估功能。该模块支持五种光源类型：点光源、聚光灯、面光源、远距光源和背景光（环境光/HDRI），以及网格发光体（三角形光源）。

模块的核心功能包括：基于光源树（Light Tree）的重要性采样、光源分布表（CDF）采样、多重重要性采样（MIS）权重计算，以及光源链接（Light Linking）和阴影链接（Shadow Linking）等高级特性。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `light.h` | 核心调度 | 光源采样与求交的顶层调度器，根据光源类型分发到具体实现；包含光源链接匹配逻辑 |
| `sample.h` | 采样入口 | 光源采样主入口：着色器评估、MIS 权重计算、下一事件估计（NEE）的完整采样流程 |
| `tree.h` | 光源树 | 光源树重要性采样实现，基于 Conty Estevez & Kulla 的自适应树分裂算法 |
| `distribution.h` | 分布采样 | 基于 CDF 的光源分布表采样，作为光源树的替代方案 |
| `common.h` | 公共工具 | 光源采样的公共工具函数 |
| `area.h` | 面光源 | 矩形/圆盘面光源的采样、求交和 PDF 计算 |
| `spot.h` | 聚光灯 | 聚光灯的采样、求交和 PDF 计算，包含锥形衰减 |
| `point.h` | 点光源 | 球形点光源的采样、求交和 PDF 计算 |
| `distant.h` | 远距光源 | 平行光/太阳光的采样和求交，支持有限角径 |
| `background.h` | 背景光 | 环境贴图光源的采样，基于亮度 CDF 的重要性采样 |
| `triangle.h` | 三角形光源 | 网格发光体的三角形光源采样和 PDF 计算 |

## 核心类与数据结构

- **`LightSample`**：光源采样结果，包含采样点位置 `P`、方向 `D`、法线 `Ng`、PDF `pdf`、评估因子 `eval_fac`、发射体选择 PDF `pdf_selection`、光源类型和着色器 ID 等。
- **`KernelLight`**：内核光源数据结构，存储每个光源的类型、着色器 ID、对象 ID、最大反弹次数、焦散标记以及类型特定参数。
- **`KernelLightTreeEmitter`**：光源树发射体节点，包含光源/三角形 ID、对象 ID 和着色器标记。
- **`KernelLightDistribution`**：光源分布表条目，包含图元 ID、对象 ID 和着色器标记。
- **`KernelBoundingBox`**：光源树节点包围盒，用于重要性计算中的立体角估计。
- **`LightType`**：光源类型枚举（`LIGHT_POINT`、`LIGHT_SPOT`、`LIGHT_AREA`、`LIGHT_DISTANT`、`LIGHT_BACKGROUND`、`LIGHT_TRIANGLE`）。
- **`BsdfEval`**：BSDF 评估结果，按漫射和光泽分量存储，用于渲染通道分离。

## 内核函数入口

### 光源采样
- **`light_sample<in_volume_segment>()`**（单光源版）：对指定光源进行位置采样，根据类型分发到 `point_light_sample()`、`spot_light_sample()`、`area_light_sample()`、`distant_light_sample()` 或 `background_light_sample()`。
- **`light_sample<in_volume_segment>()`**（场景级版）：从光源树或分布表选择发射体后，完成位置采样和 PDF 计算，支持光源链接过滤。
- **`light_sample_shader_eval()`**：对采样的光源评估着色器，获取发射光谱，处理常量发射和非常量发射（需运行 SVM）两种情况。

### 光源求交
- **`lights_intersect()`**：主路径光源求交，遍历所有解析光源（点、聚光灯、面光源），查找最近命中，用于前向路径的 MIS。
- **`lights_intersect_shadow_linked()`**：阴影链接专用光源求交，额外处理远距光源，使用蓄水池采样支持多次命中。
- **`light_sample_from_intersection()`**：从光线-光源求交结果反推光源采样参数和 PDF。

### 光源树
- **`light_tree_cos_bound_subtended_angle()`**：计算光源树节点包围盒对着色点所张的最小包围球余弦角。
- **`compute_v()`**：计算光源树重要性评估中的方向向量，用于方向性光源的包围锥测试。

### 光源链接
- **`light_link_light_match()`**：检查光源发射体是否匹配接收器对象的光链接集。
- **`light_link_object_match()`**：检查对象发射体是否匹配接收器对象的光链接集。

## GPU 兼容性

该模块完全兼容 GPU：

- 所有函数使用 `ccl_device`/`ccl_device_inline`/`ccl_device_noinline`/`ccl_device_forceinline` 宏。
- 光源链接使用 `uint64_t` 位集和位运算，支持最多 64 个光链接集。
- 光源树通过 `__LIGHT_TREE__` 编译特性开关控制。
- 光源链接通过 `__LIGHT_LINKING__`、阴影链接通过 `__SHADOW_LINKING__` 特性宏控制。
- MNEE（流形下一事件估计）通过 `__MNEE__` 特性宏控制焦散光源过滤。

## 依赖关系

### 上游依赖（本模块依赖）
- `kernel/globals.h`：全局内核数据访问（`kernel_data_fetch`）
- `kernel/types.h`：基础类型定义
- `kernel/geom/object.h`：对象变换与元数据（lightgroup、光链接集成员关系）
- `kernel/geom/shader_data.h`：`ShaderData` 初始化（`shader_setup_from_sample`、`shader_setup_from_background`）
- `kernel/integrator/state.h`：积分器状态访问（MIS 光线对象）
- `kernel/integrator/surface_shader.h`：表面着色器评估（`surface_shader_eval`）
- `kernel/sample/lcg.h`：线性同余随机数（蓄水池采样）
- `kernel/sample/mis.h`：多重重要性采样工具函数
- `util/math_fast.h`：快速数学函数（光源树中使用）

### 下游依赖（依赖本模块）
- `kernel/integrator/shade_surface.h`：表面着色积分器，调用 NEE（下一事件估计）光源采样
- `kernel/integrator/shade_volume.h`：体积着色积分器，调用体积段光源采样
- `kernel/integrator/shade_light.h`：光源着色内核，处理前向路径命中光源时的 MIS
- `kernel/integrator/shade_background.h`：背景着色内核，处理环境光的 MIS
- `kernel/integrator/shade_dedicated_light.h`：阴影链接专用光源着色
- `kernel/integrator/intersect_closest.h`：最近求交内核，调用 `lights_intersect()`
- `kernel/integrator/mnee.h`：流形下一事件估计，通过折射表面采样光源

## 关键算法与实现细节

1. **光源树（Light Tree）重要性采样**：基于 Conty Estevez & Kulla (2018) 的论文实现，但做了重要修改。原论文在节点方差过高时遍历两个子节点（分裂），而 Cycles 不支持单着色点多光源，因此改用最小-最大贡献的均匀混合作为重要性度量。同时扩展了对远距光源的支持。

2. **光源链接（Light Linking）**：通过 64 位集合成员关系实现。每个对象持有 `light_set_membership`（属于哪些光集）和 `receiver_light_set`（接收哪个光集），通过位与运算判断匹配关系。

3. **阴影链接（Shadow Linking）**：与光源链接配合，对非全局可见的光源使用专用内核路径（`intersect_dedicated_light` + `shade_dedicated_light`）进行独立的阴影求交和着色。

4. **下一事件估计（NEE）**：`light_sample_shader_eval()` 实现完整的 NEE 流程 -- 先尝试常量发射快速路径，失败则构建 `ShaderData` 运行完整着色器评估。这对 GPU 一致性（coherence）和编译时间都有显著优化。

5. **MIS 权重**：光源采样的 PDF 由选择概率（`pdf_selection`）和位置采样 PDF（`pdf`）的乘积组成，确保与 BSDF 采样之间的 MIS 权重正确平衡。

6. **体积段特化**：通过模板参数 `in_volume_segment` 对体积段内的光源采样进行特化处理，远距和背景光源在体积段中只生成占位样本。

## 参见

- `src/kernel/integrator/shade_surface.h` - 表面着色中的直接光照采样
- `src/kernel/integrator/mnee.h` - 通过折射面的流形下一事件估计
- `src/kernel/bvh/` - BVH 加速结构（光线求交）
- `src/scene/light.cpp` - 主机端光源数据准备与光源树构建
