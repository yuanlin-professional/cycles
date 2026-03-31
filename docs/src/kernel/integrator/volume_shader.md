# volume_shader.h - 体积着色器评估与相函数操作

## 概述

`volume_shader.h` 提供了体积着色器评估和体积相函数操作的完整功能集合。该文件负责体积闭包的合并、相函数的采样与评估、运动模糊下的体积自平流处理，以及完整的体积着色器评估入口。它是体积渲染管线中着色器评估层的核心组件。

## 核心函数

### 闭包合并
- **`volume_shader_merge_closures`**: 合并 ShaderData 中相同的体积散射闭包，节省闭包槽位。在堆叠体积场景中特别重要，因为多个体积着色器的闭包会累积到同一数组中。

### 闭包复制
- **`volume_shader_copy_phases`**: 从 ShaderData 中提取所有体积散射闭包，复制到紧凑的 `ShaderVolumePhases` 结构体，用于后续散射采样。

### 路径引导
- **`volume_shader_prepare_guiding`**: 初始化体积路径引导。当有多个相函数时按权重随机选择一个用于引导，调整其采样权重以包含引导概率，然后调用 `guiding_phase_init` 初始化引导缓存查询。

### 相函数评估
- **`volume_shader_phase_pick`**: 按采样权重随机选择一个体积相函数闭包，使用储层采样实现。
- **`_volume_shader_phase_eval_mis`**: 内部MIS评估函数，遍历所有相函数闭包累加PDF和评估值。
- **`volume_shader_phase_eval`** (两个重载): 单闭包评估和多闭包MIS评估，后者集成了路径引导的PDF组合。

### 相函数采样
- **`volume_shader_phase_guided_sample`**: 引导采样实现，在引导分布和体积相函数之间按引导概率切换采样源，计算混合PDF。
- **`volume_shader_phase_sample`**: 经典（无引导）相函数采样，返回采样方向、PDF和粗糙度。

### 运动模糊
- **`volume_shader_motion_blur`**: 使用一阶半拉格朗日对流方案处理体积运动模糊。从速度属性读取速度场，执行两步对流以估算在给定时间点的体积量：先查找速度、计算对流位置，再在对流位置查找速度并重新计算最终位置。

### 体积着色器评估
- **`volume_shader_eval_entry`**: 评估单个体积栈条目的着色器。设置着色器数据、检查可见性、处理对象变换和运动模糊，调用 SVM 或 OSL 评估着色器节点。
- **`volume_shader_eval`**: 遍历体积栈中的所有条目，累积闭包到 ShaderData。对非阴影路径在每个条目后执行闭包合并以避免超出闭包限制。

## 依赖关系

- **内部头文件**:
  - `kernel/closure/volume.h` - 体积闭包类型和相函数
  - `kernel/geom/attribute.h` - 几何属性访问
  - `kernel/geom/shader_data.h` - 着色器数据设置
  - `kernel/svm/svm.h` - SVM 着色器虚拟机（条件编译）
  - `kernel/osl/osl.h` - OSL 着色器（条件编译）
  - `kernel/film/light_passes.h` - 光照通道
  - `kernel/integrator/guiding.h` - 路径引导
  - `kernel/integrator/volume_stack.h` - 体积栈读取
- **被引用**: `shade_volume.h`（体积着色内核）、`bake/bake.h`（烘焙）

## 实现细节 / 关键算法

1. **闭包合并策略**: 通过比较相函数参数（`volume_phase_equal`）找到相同的闭包并合并权重。合并是原地操作，后续闭包前移填补空位。这在多体积堆叠场景中关键，因为每个体积贡献的闭包数可能累积超过 `MAX_VOLUME_CLOSURE` 限制。

2. **半拉格朗日运动模糊**: 体积运动模糊使用自平流方案：在时间 T 查找位置 x 处的体积量 φ(x, T)，等价于在时间 t=0 查找位置 x - (T-t)*u(x,T) 处的体积量。速度场 u(x,T) 本身也通过自平流估算，执行两次查找。始终使用线性插值查找速度。

3. **阴影路径优化**: 阴影光线和终止路径不需要存储闭包（`max_closures = 0`），仅计算消光和发光。这减少了内存使用和着色器评估开销。

4. **体积栈遍历**: `volume_shader_eval` 遍历体积栈直到遇到 `SHADER_NONE` 哨兵值。对于非阴影路径，在每处理一个栈条目后执行闭包合并，防止闭包溢出。

5. **引导采样权重调整**: 使用引导时，被选中用于引导的相函数闭包的 `sample_weight` 被乘以 `volume_guiding_probability`，使得后续的相函数选择自然地偏向引导分布。

## 关联文件

- `shade_volume.h` - 体积着色的完整流程
- `volume_stack.h` - 体积栈管理
- `kernel/closure/volume.h` - Henyey-Greenstein 等相函数实现
- `surface_shader.h` - 表面着色器的对应实现
