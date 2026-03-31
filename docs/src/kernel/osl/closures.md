# closures.cpp + closures_setup.h + closures_template.h - 开放着色语言(OSL)闭包系统

## 概述

这组文件构成了 Cycles 渲染器中开放着色语言(OSL)闭包/双向散射分布函数(BSDF)系统的核心。`closures_template.h` 使用 X-Macro 模式定义了所有支持的闭包类型及其参数;`closures_setup.h` 为每种闭包类型提供设置(setup)函数,将开放着色语言(OSL)闭包参数转换为 Cycles 内部的双向散射分布函数(BSDF)数据结构;`closures.cpp` 负责向 OSL ShadingSystem 注册所有闭包类型,并实现了表面、体积、位移和相机着色器的执行入口,以及闭包树的扁平化处理逻辑。

## 类与结构体

### closures_template.h 定义的闭包结构体

通过 X-Macro 模式(`OSL_CLOSURE_STRUCT_BEGIN/END/MEMBER`),在不同包含上下文中生成不同代码。所有闭包结构体按 8 字节对齐,包含 `label` 字段。主要闭包类型包括:

**漫反射类**:
- `DiffuseClosure` - Lambert 漫反射(参数: N)
- `OrenNayarClosure` - Oren-Nayar 漫反射(参数: N, roughness),已弃用
- `OrenNayarDiffuseBSDFClosure` - Oren-Nayar 漫反射 BSDF(参数: N, albedo, roughness)
- `BurleyDiffuseBSDFClosure` - Burley 漫反射 BSDF(参数: N, albedo, roughness)
- `TranslucentClosure` - 半透明(参数: N)

**光泽反射/折射类**:
- `ReflectionClosure` - 理想镜面反射(参数: N)
- `RefractionClosure` - 理想折射(参数: N, ior)
- `TransparentClosure` - 透明(无参数)
- `DielectricBSDFClosure` - 电介质 BSDF(参数: N, T, reflection_tint, transmission_tint, alpha_x, alpha_y, ior, distribution, thinfilm_thickness, thinfilm_ior)
- `ConductorBSDFClosure` - 导体 BSDF(参数: N, T, alpha_x, alpha_y, ior, extinction, distribution, thinfilm_thickness, thinfilm_ior)
- `GeneralizedSchlickBSDFClosure` - 广义 Schlick BSDF(参数: N, T, reflection_tint, transmission_tint, alpha_x, alpha_y, f0, f90, exponent, distribution, thinfilm_thickness, thinfilm_ior)
- `MicrofacetClosure` - 微面元(参数: distribution, N, T, alpha_x, alpha_y, ior, refract)
- `MicrofacetF82TintClosure` - 带 F82 色调的微面元(参数: distribution, N, T, alpha_x, alpha_y, f0, f82, thinfilm_thickness, thinfilm_ior)
- `MicrofacetMultiGGXGlassClosure` - 多散射 GGX 玻璃(参数: N, alpha_x, ior, color)
- `MicrofacetMultiGGXClosure` - 多散射 GGX 各向异性(参数: N, T, alpha_x, alpha_y, color)
- `RayPortalBSDFClosure` - 光线入口 BSDF(参数: position, direction)

**布料/天鹅绒类**:
- `AshikhminVelvetClosure` - Ashikhmin 天鹅绒(参数: N, sigma)
- `SheenClosure` - 光泽(参数: N, roughness)
- `SheenBSDFClosure` - 光泽 BSDF(参数: N, albedo, roughness)

**卡通着色类**:
- `DiffuseToonClosure` - 漫反射卡通(参数: N, size, smooth)
- `GlossyToonClosure` - 光泽卡通(参数: N, size, smooth)

**发射/背景类**:
- `GenericEmissiveClosure` - 发射(无参数)
- `GenericBackgroundClosure` - 背景(无参数)
- `UniformEDFClosure` - 均匀发射分布(参数: emittance)
- `HoldoutClosure` - 遮罩(无参数)

**渐变类**:
- `DiffuseRampClosure` - 漫反射渐变(参数: N, colors[8])
- `PhongRampClosure` - Phong 渐变(参数: N, exponent, colors[8])

**次表面散射类**:
- `BSSRDFClosure` - 次表面散射(参数: method, N, radius, albedo, roughness, ior, anisotropy)
- `SubsurfaceBSSRDFClosure` - 次表面散射 BSSRDF(参数: N, albedo, radius/transmission, anisotropy)

**毛发类**:
- `HairReflectionClosure` - 毛发反射(参数: N, roughness1, roughness2, T, offset)
- `HairTransmissionClosure` - 毛发透射(参数: N, roughness1, roughness2, T, offset)
- `ChiangHairClosure` - Chiang 毛发模型(参数: N, sigma, v, s, m0_roughness, alpha, eta)
- `HuangHairClosure` - Huang 毛发模型(参数: N, sigma, roughness, tilt, eta, aspect_ratio, r_lobe, tt_lobe, trt_lobe)

**体积类**:
- `VolumeAbsorptionClosure` - 体积吸收(无参数)
- `VolumeHenyeyGreensteinClosure` - Henyey-Greenstein 相函数(参数: g)
- `VolumeFournierForandClosure` - Fournier-Forand 相函数(参数: B, IOR)
- `VolumeDraineClosure` - Draine 相函数(参数: g, alpha)
- `VolumeRayleighClosure` - Rayleigh 散射(无参数)

**特殊类型**:
- `LayerClosure` - 分层闭包(参数: top, base),用于闭包的分层混合

### closures_setup.h 中的辅助结构

- 每个闭包类型对应一个 `osl_closure_<name>_setup()` 函数,负责分配 BSDF 内存并初始化参数

## 核心函数

### closures.cpp

#### `OSLRenderServices::register_closures(OSL::ShadingSystem *ss)`
- **功能**: 向 OSL ShadingSystem 注册所有支持的闭包类型。通过包含 `closures_template.h` 自动展开注册代码,另外手动注册 `layer` 闭包。

#### `osl_eval_nodes<SHADER_TYPE_SURFACE>`
- **功能**: 执行表面/背景着色器。流程:
  1. 调用 `shaderdata_to_shaderglobals` 初始化着色器全局变量
  2. 根据路径标志区分光线状态和阴影状态
  3. 若对象为 `OBJECT_NONE`,执行背景着色器
  4. 否则,先执行自动凹凸着色器(处理位移前的状态恢复),再执行表面着色器
  5. 调用 `flatten_closure_tree` 将闭包树扁平化

#### `osl_eval_nodes<SHADER_TYPE_VOLUME>`
- **功能**: 执行体积着色器。流程与表面着色器类似,但只执行体积着色器状态。

#### `osl_eval_nodes<SHADER_TYPE_DISPLACEMENT>`
- **功能**: 执行位移着色器。执行完毕后从着色器全局变量中回读修改后的位置 `P`。

#### `osl_eval_camera` (CPU 实现)
- **功能**: 执行相机着色器,通过 OSL ShadingSystem 计算光线的起点、方向及其偏导数。

### closures_setup.h

#### `osl_closure_skip(KernelGlobals kg, uint32_t path_flag, int scattering)`
- **功能**: 根据焦散选项判断是否跳过当前闭包。检查反射焦散和折射焦散的启用状态。

#### `osl_zero_albedo(float3 *layer_albedo)`
- **功能**: 将图层反照率归零。当闭包分配失败时使用,确保下层闭包不会被错误遮挡。

#### 各闭包 setup 函数 (约 40+ 个)
- **函数签名模式**: `osl_closure_<name>_setup(KernelGlobals, ShaderData*, uint32_t path_flag, float3 weight, const <Name>Closure*, float3 *layer_albedo)`
- **功能**: 为每种闭包类型分配 BSDF 内存,从闭包参数中提取数据并设置 Cycles 内部的 BSDF 结构体,然后调用对应的 `bsdf_*_setup()` 函数完成初始化。

### osl.h

#### `shaderdata_to_shaderglobals(ShaderData *sd, uint32_t path_flag, ShaderGlobals *globals)`
- **功能**: 将 Cycles 内部的 `ShaderData` 转换为开放着色语言(OSL)的 `ShaderGlobals` 结构体。映射位置、法线、UV、方向、时间等所有着色数据,并设置 raytype、backfacing 等标志位。

#### `flatten_closure_tree(KernelGlobals kg, ShaderData *sd, uint32_t path_flag, const OSLClosure *closure)`
- **功能**: 将开放着色语言(OSL)生成的闭包树扁平化为 Cycles 可直接使用的 BSDF 列表。使用栈式迭代遍历闭包树:
  - `OSL_CLOSURE_MUL_ID`: 将权重与当前闭包相乘后继续遍历
  - `OSL_CLOSURE_ADD_ID`: 将右子树入栈,继续遍历左子树
  - `OSL_CLOSURE_LAYER_ID`: 处理分层闭包,先处理顶层,再根据顶层的反照率调整底层权重
  - 其他: 调用对应的 `osl_closure_<name>_setup` 函数

## 依赖关系

### closures.cpp
- **内部头文件**:
  - `<OSL/genclosure.h>`, `<OSL/oslclosure.h>` - OSL 闭包生成和定义
  - `kernel/types.h` - 内核类型定义
  - `kernel/osl/globals.h` - OSL 全局变量
  - `kernel/osl/services.h` - OSL 渲染服务
  - `kernel/osl/camera.h` - 相机着色器接口
  - `kernel/osl/osl.h` - OSL 着色器引擎
  - `kernel/geom/attribute.h`, `kernel/geom/object.h`, `kernel/geom/primitive.h` - 几何属性访问
  - `util/math.h`, `util/param.h` - 数学和参数工具
- **被引用**: 编译为独立翻译单元,不被其他文件直接 `#include`

### closures_setup.h
- **内部头文件**:
  - `kernel/closure/alloc.h` - 闭包内存分配
  - `kernel/closure/bsdf.h` - BSDF 相关函数
  - `kernel/closure/bssrdf.h` - BSSRDF 次表面散射
  - `kernel/closure/emissive.h` - 发射闭包
  - `kernel/closure/volume.h` - 体积闭包
  - `kernel/geom/object.h` - 几何对象操作
  - `kernel/osl/types.h` - OSL 类型定义
- **被引用**:
  - `src/kernel/osl/osl.h`

### closures_template.h
- **内部头文件**: 无(纯 X-Macro 模板)
- **被引用**:
  - `src/kernel/osl/types.h` - 生成 `OSLClosureType` 枚举
  - `src/kernel/osl/closures_setup.h` - 生成闭包结构体定义
  - `src/kernel/osl/closures.cpp` - 生成注册代码和闭包参数定义
  - `src/kernel/osl/osl.h` - 在 `flatten_closure_tree` 中生成 switch-case 分支

## 实现细节 / 关键算法

### X-Macro 模式

`closures_template.h` 采用经典的 X-Macro 设计模式。该文件不包含 `#pragma once` 保护,可被多次包含。每次包含前,调用方重新定义四个宏:
- `OSL_CLOSURE_STRUCT_BEGIN(Upper, lower)` - 开始定义闭包
- `OSL_CLOSURE_STRUCT_END(Upper, lower)` - 结束定义闭包
- `OSL_CLOSURE_STRUCT_MEMBER(Upper, TYPE, type, name, key)` - 定义普通成员
- `OSL_CLOSURE_STRUCT_ARRAY_MEMBER(Upper, TYPE, type, name, key, size)` - 定义数组成员

不同上下文中的展开方式:
1. **类型定义** (types.h): 展开为枚举值 `OSL_CLOSURE_<Upper>_ID`
2. **结构体定义** (closures_setup.h): 展开为 `struct <Upper>Closure { ... }`
3. **注册代码** (closures.cpp): 展开为 `OSL::ClosureParam` 数组和 `register_closure` 调用
4. **求值分支** (osl.h): 展开为 `flatten_closure_tree` 中的 `case` 分支

### 闭包树扁平化算法

`flatten_closure_tree` 使用固定大小为 16 的栈进行迭代遍历(非递归),避免了深度不确定的闭包树导致的栈溢出风险。分层闭包(Layer)的处理逻辑:
1. 将底层闭包入栈,记录栈层级
2. 处理顶层闭包,累积其反照率
3. 栈弹出底层时,根据顶层反照率通过 `closure_layering_weight` 调整底层权重
4. 若底层完全被遮挡(权重为零),跳过底层

### 自动凹凸着色器执行

在表面着色器执行前,若存在凹凸映射,先将几何位置回退到未位移状态(通过读取 `ATTR_STD_POSITION_UNDISPLACED` 属性),执行凹凸着色器修改法线,然后恢复原始几何状态,仅保留修改后的法线用于后续着色。

### 焦散控制

`osl_closure_skip` 函数实现了基于路径标志的焦散过滤。当光线处于漫反射路径且闭包包含光泽散射时,根据集成器设置决定是否跳过反射焦散和/或折射焦散。

## 关联文件

- `src/kernel/osl/types.h` - 闭包类型枚举和基础结构体定义
- `src/kernel/osl/globals.h` - 着色器状态数组(surface_state, volume_state 等)
- `src/kernel/osl/camera.h` - 相机着色器接口
- `src/kernel/closure/bsdf.h` - Cycles 内部 BSDF 实现
- `src/kernel/closure/alloc.h` - BSDF 内存分配器
