# shader.h / shader.cpp - 着色器管理与编译调度核心

## 概述

本文件定义了 Cycles 渲染器中着色器（Shader）的核心数据结构和着色器管理器（ShaderManager）基类。`Shader` 类描述网格、灯光或背景的外观，包含表面、体积和位移三个输出通道；`ShaderManager` 是着色器虚拟机(SVM)和开放着色语言(OSL)着色器管理器的虚基类，负责着色器的编译调度、设备更新和查找表管理。着色器编译过程中会根据条件选择创建 `SVMShaderManager` 或 `OSLShaderManager`。

## 类与结构体

### Shader

- **继承**: `Node`（来自 `graph/node.h`）
- **功能**: 描述单个着色器的完整状态，包含着色器图、编译后属性标记、发射估算等。尽管只有一个着色器图(`ShaderGraph`)，但它有 Surface、Volume、Displacement 三个输出，由着色器管理器分别编译和执行。
- **关键成员**:
  - `unique_ptr<ShaderGraph> graph` — 着色器节点图
  - `has_surface` / `has_volume` / `has_displacement` — 编译后标志，指示着色器包含哪些组件
  - `has_surface_transparent` — 是否包含透明表面
  - `has_surface_bssrdf` — 是否包含次表面散射
  - `has_surface_raytrace` — 是否包含光线追踪节点
  - `has_volume_connected` — 优化前是否有体积子树连接（用于优化后仍保持表面透明）
  - `emission_estimate` — 发射强度估算值（float3）
  - `emission_sampling` — 发射采样策略（EmissionSampling 枚举）
  - `emission_is_constant` — 发射是否为常量（可跳过着色器求值）
  - `attributes` — 请求的网格属性集合（AttributeRequestSet）
  - `id` — 全局唯一着色器 ID
  - `displacement_method` — 位移方法（Bump / True / Both）
  - `volume_sampling_method` — 体积采样方法（距离/等角/多重重要性）
  - `volume_step_rate` — 体积步进速率
  - `osl_cache` — (WITH_OSL 条件编译) 每个设备的 OSL 编译缓存
- **关键方法**:
  - `estimate_emission()` — 基于着色器图估算发射强度，用于改善灯光重要性采样。支持 EmissionNode、BackgroundNode、PrincipledBsdfNode、LightFalloffNode、IESLightNode、AddClosureNode、MixClosureNode 等节点的递归估算
  - `set_graph(unique_ptr<ShaderGraph>&&)` — 设置着色器图，同时移除代理节点并计算位移哈希
  - `tag_update(Scene*)` — 标记着色器已修改，触发场景各管理器的更新标记（灯光、几何体、体积等）
  - `tag_used(Scene*)` — 当未使用的着色器突然被引用时，触发重新编译
  - `has_surface_shadow_transparency()` — 检查是否包含透明阴影
  - `need_update_geometry()` — 检查是否需要更新几何体（UV/属性/位移）
  - `has_surface_link()` — Surface 或 Displacement 输出是否有连接

### ShaderManager

- **继承**: 无（虚基类）
- **功能**: 着色器管理器的抽象基类，`SVMShaderManager` 和 `OSLShaderManager` 从此派生。负责着色器编译流水线、设备数据上传、查找表管理和色彩空间转换。
- **关键成员**:
  - `update_flags` — 更新标志位（SHADER_ADDED / SHADER_MODIFIED / UPDATE_ALL / UPDATE_NONE）
  - `unique_attribute_id` — 属性名到唯一 ID 的映射表
  - `bsdf_tables` — 双向散射分布函数(BSDF)查找表缓存
  - `thin_film_table_offset_` — 薄膜干涉查找表偏移
  - `xyz_to_r/g/b` — XYZ 到 RGB 色彩空间变换矩阵行向量
  - `rgb_to_y` — RGB 到亮度转换系数
  - `rec709_to_r/g/b` — Rec.709 到场景线性空间转换
  - `scene_linear_space` — 当前场景线性空间类型（Rec709/Rec2020/ACEScg/Unknown）
  - `thin_film_table` — 薄膜干涉预计算查找表
- **关键方法**:
  - `create(int shadingsystem)` — 工厂方法，根据着色系统类型创建 `SVMShaderManager` 或 `OSLShaderManager`
  - `device_update_pre()` — 设备更新前处理：分配着色器 ID，预处理着色器图（finalize），检测体积着色器
  - `device_update_post()` — 设备更新后处理：调用派生类的 `device_update_specific()`，将数据拷贝到设备
  - `device_update_common()` — 通用设备更新：设置内核着色器标志、加载 BSDF 查找表、设置色彩空间矩阵、薄膜表
  - `device_free_common()` — 通用设备释放：清理查找表和着色器数据
  - `add_default(Scene*)` — 添加默认着色器（default_surface 使用 PrincipledBsdf、default_light 使用 Emission、default_background 为空、default_empty 为空、default_volume 使用 PrincipledVolume）
  - `get_kernel_features(Scene*)` — 收集所有着色器使用的内核特性标志
  - `get_attribute_id(ustring/AttributeStandard)` — 获取属性的全局唯一 ID
  - `get_shader_id(Shader*, bool smooth)` — 获取传递给内核的着色器 ID（含平滑法线标志）
  - `init_xyz_transforms()` — 初始化 XYZ 色彩空间变换矩阵，支持通过 OpenColorIO 获取
  - `compute_thin_film_table()` — 计算薄膜干涉查找表（基于 Belcour & Barla 2017 论文的傅里叶变换方法）
  - `ensure_bsdf_table()` — 确保 BSDF 查找表已上传到设备
  - `get_cryptomatte_materials()` — 生成 Cryptomatte 材质清单（JSON 格式）
  - `linear_rgb_to_gray()` — 线性 RGB 转灰度
  - `rec709_to_scene_linear()` — Rec.709 转场景线性空间

## 枚举类型

- `ShadingSystem` — 着色系统选择：`SHADINGSYSTEM_OSL` / `SHADINGSYSTEM_SVM`
- `VolumeSampling` — 体积采样方法：距离 / 等角 / 多重重要性采样
- `VolumeInterpolation` — 体积插值方法：线性 / 三次
- `DisplacementMethod` — 位移方法：Bump / True / Both

## 核心函数

- `output_estimate_emission(ShaderOutput*, bool&)` — 静态递归函数，沿着色器图估算发射强度。对简单情况（EmissionNode、BackgroundNode 等）直接计算，对闭包混合节点递归处理。返回估算的 float3 发射值，同时通过引用参数返回是否为常量发射。

## 依赖关系

- **内部头文件**:
  - `kernel/types.h` — 内核类型定义
  - `scene/attribute.h` — 属性请求集合
  - `graph/node.h` — 节点基类
  - `util/map.h`, `util/param.h`, `util/string.h`, `util/thread.h`, `util/types.h`, `util/unique_ptr.h`
- **实现文件额外依赖**:
  - `scene/osl.h`, `scene/svm.h` — 派生着色器管理器
  - `scene/shader_graph.h`, `scene/shader_nodes.h` — 着色器图和节点
  - `scene/tables.h` — 查找表管理
  - `scene/shader.tables` — 预计算 BSDF 查找表数据
  - `util/murmurhash.h` — Cryptomatte 哈希
  - OpenColorIO（可选，WITH_OCIO）
- **被引用**: `scene/scene.h`, `scene/svm.h`, `scene/osl.h`, `scene/shader_graph.cpp`, `scene/shader_nodes.cpp`, `scene/light.cpp`, `scene/mesh.h`, `scene/geometry*.cpp`, `scene/background.cpp`, `scene/integrator.cpp`, `scene/bake.cpp`, `session/session.h`, `hydra/*.cpp`, `app/cycles_xml.cpp` 等 23 个文件

## 实现细节 / 关键算法

### 发射估算算法
`estimate_emission()` 通过 `output_estimate_emission()` 递归遍历着色器图，从 Surface 输出开始向上追踪：
1. 对 EmissionNode / BackgroundNode：获取 Color 和 Strength 输入值
2. 对 PrincipledBsdfNode：获取 Emission Color 和 Emission Strength，但标记为非常量
3. 对 LightFalloffNode / IESLightNode：获取 Strength
4. 对 AddClosureNode：两个输入的估算值相加
5. 对 MixClosureNode：根据 Fac 值加权混合
6. 自动转换节点（from_auto_conversion）的发射权重降低为 0.1 倍
7. 根据估算结果决定发射采样策略（NONE / FRONT_BACK / 用户指定）

### 薄膜干涉查找表
基于 Belcour & Barla (2017) 论文 "A Practical Extension to Microfacet Theory for the Modeling of Varying Iridescence"。预计算 XYZ 颜色匹配函数的傅里叶变换，运行时乘以 XYZ-to-RGB 矩阵得到 RGB 查找表。利用重采样和 FFT 的线性性，将预计算数据与运行时色彩空间矩阵结合。

### 色彩空间初始化
`init_xyz_transforms()` 默认使用 ITU-BT.709，在有 OpenColorIO 配置时优先使用 `aces_interchange` 或 `XYZ` 角色获取准确的 XYZ-to-RGB 变换矩阵。支持检测 Rec709、Rec2020、ACEScg 场景线性空间。

### shader.tables 文件
`shader.tables` 是一个以 `.tables` 为扩展名的 C++ 源文件（故意使用非标准扩展名以跳过 clang-format 格式化），通过 `#include "scene/shader.tables"` 在 `shader.cpp` 中直接包含。它包含以下预计算的双向散射分布函数(BSDF)反照率查找表（静态 float 数组）：
- `table_ggx_E[1024]` — GGX 微表面分布的方向反照率 E
- `table_ggx_Eavg[32]` — GGX 平均反照率 Eavg
- `table_ggx_glass_E[4096]` — GGX 玻璃方向反照率
- `table_ggx_glass_Eavg` — GGX 玻璃平均反照率
- `table_ggx_glass_inv_E` / `table_ggx_glass_inv_Eavg` — GGX 玻璃逆反照率
- `table_sheen_ltc` — 光泽(Sheen)线性变换余弦(LTC)表
- `table_ggx_gen_schlick_ior_s` / `table_ggx_gen_schlick_s` — GGX 广义 Schlick 近似表
- `table_thin_film_cmf` — 薄膜干涉颜色匹配函数预计算数据

这些表用于运行时能量守恒补偿和多散射近似计算。

## 关联文件

- `src/scene/shader_graph.h` / `shader_graph.cpp` — 着色器图定义
- `src/scene/shader_nodes.h` / `shader_nodes.cpp` — 着色器节点定义
- `src/scene/svm.h` / `svm.cpp` — 着色器虚拟机(SVM)管理器和编译器
- `src/scene/osl.h` / `osl.cpp` — 开放着色语言(OSL)管理器和编译器
- `src/scene/constant_fold.h` / `constant_fold.cpp` — 常量折叠
- `src/scene/shader.tables` — 预计算 BSDF 查找表数据文件
- `src/scene/tables.h` — 查找表管理接口
