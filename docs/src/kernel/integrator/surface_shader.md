# surface_shader.h - 表面着色器评估与 BSDF 操作

## 概述

`surface_shader.h` 提供了表面着色器评估和双向散射分布函数(BSDF)操作的核心功能集合。该文件涵盖了从着色器节点评估、闭包过滤、BSDF采样与评估，到路径引导集成和各种渲染通道提取的完整表面着色工具链。它是表面着色流程中最底层的接口文件，被几乎所有涉及表面交互的模块引用。

## 核心函数

### 路径引导
- **`surface_shader_average_sample_weight_squared_roughness`**: 计算所有 BSDF/BSSRDF 闭包的加权平均粗糙度平方值。
- **`surface_shader_prepare_guiding`**: 初始化表面路径引导：计算漫反射比例、BSSRDF比例、平均粗糙度，并根据引导类型（Product MIS / RIS / Roughness）设置采样概率。

### 闭包准备与过滤
- **`surface_shader_prepare_closures`**: 过滤闭包（根据调试过滤标志剔除漫反射/光泽/透射/透明闭包）并执行光泽模糊（filter glossy），基于最小光线PDF对BSDF进行粗糙度模糊以减少焦散噪声。

### BSDF 评估
- **`_surface_shader_exclude`**: 根据灯光着色器标志判断是否排除特定闭包类型。
- **`_surface_shader_bsdf_eval_mis`**: BSDF 多重重要性采样评估的内部函数，遍历所有闭包累加BSDF响应和PDF。
- **`surface_shader_bsdf_eval`**: 给定方向评估BSDF值和PDF，支持灯光着色器的排除标志。
- **`surface_shader_bsdf_eval_pdfs`**: 评估BSDF PDF，用于MIS权重计算。

### BSDF/BSSRDF 采样
- **`surface_shader_bsdf_bssrdf_pick`**: 按采样权重随机选择一个 BSDF 或 BSSRDF 闭包。
- **`surface_shader_bsdf_sample_closure`**: 从选定闭包采样出射方向、评估BSDF值，返回路径标签和采样粗糙度/折射率。
- **`surface_shader_bsdf_guided_sample_closure_mis`**: 引导采样的Product MIS实现，在引导分布和BSDF之间进行MIS。
- **`surface_shader_bsdf_guided_sample_closure_ris`**: 引导采样的RIS（重采样重要性采样）实现。
- **`surface_shader_bsdf_guided_sample_closure`**: 引导采样的顶层调度函数，根据配置选择MIS或RIS方法。

### 渲染通道提取
- **`surface_shader_average_roughness`**: 计算加权平均粗糙度。
- **`surface_shader_transparency`**: 计算透明闭包的总权重。
- **`surface_shader_alpha`**: 计算表面不透明度（1 - transparency）。
- **`surface_shader_diffuse`** / **`surface_shader_glossy`** / **`surface_shader_transmission`**: 分别提取漫反射/光泽/透射的总闭包权重。
- **`surface_shader_average_normal`**: 计算加权平均法线。
- **`surface_shader_ao`**: 计算环境光遮蔽权重和法线。
- **`surface_shader_bssrdf_normal`**: 提取BSSRDF闭包的凹凸贴图法线。

### 发光与背景
- **`surface_shader_constant_emission`**: 检查着色器是否为常量发光。
- **`surface_shader_background`**: 获取背景着色器的发光。
- **`surface_shader_emission`**: 获取表面自发光。
- **`surface_shader_apply_holdout`**: 应用holdout（镂空）并返回holdout权重。

### 着色器评估
- **`surface_shader_eval<node_feature_mask>`**: 模板化的表面着色器完整评估入口，调用SVM或OSL评估着色器节点。

## 依赖关系

- **内部头文件**:
  - `kernel/closure/bsdf.h` - BSDF 闭包实现
  - `kernel/closure/emissive.h` - 发光闭包
  - `kernel/film/light_passes.h` - 光照通道写入
  - `kernel/integrator/guiding.h` - 路径引导
  - `kernel/svm/svm.h` - SVM 着色器虚拟机（条件编译）
  - `kernel/osl/osl.h` - OSL 着色器（条件编译）
- **被引用**: `shade_surface.h`, `shade_background.h`, `shade_shadow.h`, `subsurface.h`, `light/sample.h`, `film/data_passes.h`, `bake/bake.h`

## 实现细节 / 关键算法

1. **闭包过滤系统**: `prepare_closures` 支持按类型剔除闭包（用于调试渲染通道隔离），并将透明闭包转换为holdout闭包。光泽模糊基于 `filter_glossy * min_ray_pdf` 计算模糊粗糙度，对间接光路径上的光泽BSDF进行平滑。

2. **路径引导Product MIS**: 在引导分布和BSDF分布之间使用Product MIS策略，引导概率乘以漫反射比例确保仅对漫反射分量应用引导。粗糙度低于阈值时禁用引导以避免在镜面反射上引入噪声。

3. **路径引导RIS**: 从引导分布和BSDF各生成一个候选样本，通过RIS权重选择最终样本。RIS权重基于目标分布（乘积）与提议分布的比值。

4. **BSDF验证（调试模式）**: `WITH_CYCLES_DEBUG` 启用时，`surface_shader_validate_bsdf_sample` 验证采样后的标签和粗糙度/折射率与 `bsdf_label`/`bsdf_roughness_eta` 函数的返回值一致。

5. **MNEE交互**: 当 MNEE 标志有效时，光泽模糊被跳过以保持焦散的精确反射特性。

## 关联文件

- `shade_surface.h` - 调用BSDF采样和直接光照评估
- `shade_background.h` - 调用背景着色器评估
- `kernel/closure/bsdf.h` - 各具体BSDF类型的实现
- `kernel/svm/svm.h` - SVM着色器节点评估引擎
