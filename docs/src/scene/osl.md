# osl.h / osl.cpp - 开放着色语言(OSL)着色器管理器与编译器

## 概述

本文件实现了 Cycles 渲染器中基于开放着色语言(OSL)的着色器编译和管理系统。包含三个核心组件：`OSLManager` 负责 OSL 运行时的生命周期管理（纹理系统、着色系统、着色器加载和 JIT 编译）；`OSLShaderManager` 继承自 `ShaderManager`，是 OSL 后端的着色器管理器；`OSLCompiler` 负责将着色器图编译为 OSL 着色器组。大部分代码在 `#ifdef WITH_OSL` 条件编译保护下，仅在启用 OSL 支持时编译。

## 类与结构体

### OSLShaderInfo

- **功能**: OSL 着色器的元数据信息，用于自动检测闭包特性（发射、透明、次表面散射）
- **关键成员**:
  - `query` — OSL 着色器查询对象（OSL::OSLQuery）
  - `has_surface_emission` — 是否包含表面发射
  - `has_surface_transparent` — 是否包含透明表面
  - `has_surface_bssrdf` — 是否包含次表面散射
- **条件编译**: 仅在 WITH_OSL 下可用

### OSLManager

- **继承**: 无
- **功能**: 管理 OSL 运行时系统的完整生命周期，包括纹理系统(TextureSystem)、着色系统(ShadingSystem)、着色器编译和加载。使用共享引用计数管理跨渲染实例的资源。
- **关键成员**:
  - `device_` — 关联的设备指针
  - `loaded_shaders` — 已加载着色器的缓存（hash -> OSLShaderInfo）
  - `ts` — 纹理系统的本地引用（shared_ptr<OSL::TextureSystem>）
  - `ss_map` — 按设备类型索引的着色系统映射（map<DeviceType, shared_ptr<OSL::ShadingSystem>>）
  - `need_update_` — 是否需要更新
- **关键方法**:
  - `device_update_pre()` — 设备更新前处理：初始化着色系统，注册内置纹理（@ao, @bevel），设置 OIIO 纹理系统
  - `device_update_post()` — 设备更新后处理：编译相机着色器，执行 greedyjit 优化（`optimize_all_groups()`），加载 OSL 内核
  - `device_free()` — 释放设备上的 OSL 资源，清理纹理引用
  - `osl_compile()` — 编译 OSL 源文件为字节码（.oso）
  - `osl_query()` — 查询 OSL 着色器信息
  - `shader_test_loaded()` / `shader_load_bytecode()` / `shader_load_filepath()` — 着色器加载接口
  - `shader_loaded_info()` — 获取已加载着色器的信息
  - `shading_system_init()` — 初始化共享着色系统（支持 Rec709/Rec2020/ACEScg 色彩空间）
  - `get_shading_system()` / `get_texture_system()` — 获取着色/纹理系统
  - `foreach_osl_device()` — 遍历所有具有 OSL 支持的设备
  - `texture_system_init()` / `texture_system_free()` — 纹理系统生命周期（全局共享）
  - `shading_system_free()` — 着色系统释放
  - `tag_update()` / `need_update()` — 更新标记管理

### OSLShaderManager

- **继承**: `ShaderManager`
- **功能**: 开放着色语言(OSL)后端的着色器管理器
- **条件编译**: 仅在 WITH_OSL 下可用
- **关键方法**:
  - `use_osl()` — 始终返回 true
  - `get_attribute_id()` — 获取属性 ID（OSL 使用不同的属性 ID 方案）
  - `device_update_specific()` — OSL 特有的设备更新逻辑
  - `device_free()` — 释放 OSL 设备资源
  - `osl_node()` — 静态方法，使用 OSLQuery 从文件或字节码创建 OSLNode
  - `osl_image_slots()` — 获取 OSL 服务使用的图像插槽

### OSLCompiler

- **继承**: 无
- **功能**: 将着色器图(ShaderGraph)编译为 OSL 着色器组(ShaderGroup)。遍历着色器图中的节点，将每个节点翻译为对应的 OSL 着色器调用，并建立参数连接。
- **关键成员**:
  - `ss` — OSL 着色系统指针
  - `services` — OSL 渲染服务
  - `current_group` — 当前正在构建的着色器组
  - `device` — 目标设备
  - `current_type` — 当前编译的着色器类型（Surface/Volume/Displacement/Bump）
  - `current_shader` — 当前编译的着色器
  - `texture_shared_unique_id` — 纹理共享唯一 ID（静态原子计数器）
- **关键方法**:
  - `compile(Shader*)` — 编译完整着色器（Surface + Volume + Displacement + Bump）
  - `add(ShaderNode*, name, isfilepath)` — 添加一个着色器节点到着色器组
  - `parameter(name, value)` — 设置参数（多种重载：float/float3/int/string/Transform/array 等）
  - `parameter_color/vector/normal/point()` — 类型特定的参数设置
  - `parameter_attribute()` — 设置属性参数
  - `parameter_texture()` — 设置纹理参数（文件名或 ImageHandle）
  - `output_type()` — 当前输出类型
  - `compile_type()` — 编译特定着色器类型（内部方法）
  - `find_dependencies()` / `generate_nodes()` — 依赖分析和节点生成

## 核心函数

### 全局共享资源管理
OSL 使用全局共享的纹理系统和着色系统，通过引用计数和互斥锁管理：
- `ts_shared` — 共享纹理系统（跨渲染实例复用以节省内存）
- `ss_shared` — 按设备类型分别共享的着色系统
- 使用互斥锁 `ts_shared_mutex` / `ss_shared_mutex` 保护并发访问

### 着色系统初始化
`shading_system_init()` 创建 OSL ShadingSystem 时设置：
- `lockgeom=1` — 锁定几何体
- `commonspace="world"` — 公共空间为世界空间
- `greedyjit=1` — 贪心即时编译模式
- 色彩空间配置（Rec709/Rec2020/ACEScg）
- 光线类型注册（camera/reflection/refraction/diffuse/glossy/shadow 等）
- 注册自定义闭包

### 相机着色器
`device_update_post()` 支持编译 OSL 相机着色器脚本，输出位置(P)、方向(D)、通量(throughput)及其导数。支持显式导数输出和自动导数计算两种模式。

## 依赖关系

- **内部头文件**:
  - `device/device.h` — 设备接口
  - `scene/shader.h`, `scene/shader_graph.h`, `scene/shader_nodes.h` — 着色器系统
  - `util/array.h`, `util/set.h`, `util/string.h`
  - OSL 库头文件: `OSL/llvm_util.h`, `OSL/oslcomp.h`, `OSL/oslexec.h`, `OSL/oslquery.h`
- **实现文件额外依赖**:
  - `scene/background.h`, `scene/camera.h`, `scene/colorspace.h`, `scene/light.h`, `scene/scene.h`, `scene/stats.h`
  - `kernel/osl/globals.h`, `kernel/osl/services.h` — OSL 内核端接口
  - `util/aligned_malloc.h`, `util/log.h`, `util/md5.h`, `util/path.h`, `util/progress.h`, `util/task.h`
- **被引用**: `scene/shader.cpp`, `scene/shader_nodes.cpp`, `scene/scene.cpp`, `scene/geometry*.cpp`, `scene/camera.cpp`, `app/cycles_xml.cpp` 等 8 个文件

## 实现细节 / 关键算法

### Greedy JIT 优化
`device_update_post()` 在锁内调用 `ss->optimize_all_groups()`，对所有着色器组执行即时编译优化。这种策略可能会优化一些未实际使用的着色器组，但避免了渲染时在 TLS（线程本地存储）上分配数据，从而避免了任务调度器线程的 TLS 清理问题和内存释放顺序问题。

### OSL 编译器工作流程
1. 遍历着色器图中的节点
2. 使用 `find_dependencies()` 确定依赖关系
3. 按拓扑序调用 `generate_nodes()` 生成 OSL 着色器
4. 每个节点调用 `node->compile(osl_compiler)`，节点内部通过 compiler 的 `add()` 和 `parameter()` 方法向着色器组添加着色器和参数
5. 分别为 Surface、Volume、Displacement、Bump 编译四个着色器组

### Windows 路径处理
由于 Cycles 内部使用 UTF-8 编码存储路径，而 OSL 使用 ANSI 函数，在 Windows 上需要通过 `string_to_ansi()` 转换路径。这意味着 OSL 在包含多字节字符的着色器路径下可能无法正常工作。

### LLVM 清理
由于 LLVM 和 OSL 全局析构函数执行顺序不确定（尤其在 Windows 上），使用 `OSL::pvt::LLVM_Util::Cleanup()` 在进程退出前主动清理 LLVM 资源，防止崩溃。

## 关联文件

- `src/scene/shader.h` / `shader.cpp` — ShaderManager 基类
- `src/scene/shader_graph.h` / `shader_graph.cpp` — 着色器图（编译输入）
- `src/scene/shader_nodes.h` / `shader_nodes.cpp` — 节点类型（每个节点有 OSL compile 方法）
- `src/scene/svm.h` / `svm.cpp` — SVM 编译器（替代后端）
- `src/kernel/osl/globals.h` — OSL 全局数据结构
- `src/kernel/osl/services.h` — OSL 渲染服务接口
