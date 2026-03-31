# CMakeLists.txt — 场景模块构建配置

## 概述

`CMakeLists.txt` 是 Cycles 场景模块（`cycles_scene`）的 CMake 构建脚本，定义了源文件列表、头文件列表、包含目录和链接库依赖。场景模块是 Cycles 的核心组件，负责管理所有渲染数据（几何体、材质、灯光、相机等）。

## 构建目标

- **库名称**: `cycles_scene`
- **构建方式**: `cycles_add_library()` — Cycles 自定义的库构建宏

## 源文件列表 (SRC)

场景模块包含以下 .cpp 源文件，按功能分组：

### 几何体相关
- `geometry.cpp` — 几何体基类与管理器主逻辑
- `geometry_attributes.cpp` — 属性上传到设备
- `geometry_bvh.cpp` — 层次包围体(BVH)构建与打包
- `geometry_mesh.cpp` — 网格/曲线/点云数据打包
- `mesh.cpp` — 三角网格
- `mesh_displace.cpp` — 网格位移着色器求值
- `mesh_subdivision.cpp` — 曲面细分
- `hair.cpp` — 毛发几何体
- `curves.cpp` — 曲线工具函数
- `pointcloud.cpp` — 点云几何体
- `volume.cpp` — 体积几何体与管理器

### 属性与对象
- `attribute.cpp` — 属性系统
- `object.cpp` — 场景对象与管理器
- `particles.cpp` — 粒子系统

### 渲染管线
- `background.cpp` — 背景/环境
- `camera.cpp` — 相机
- `film.cpp` — 胶片/输出设置
- `integrator.cpp` — 积分器
- `light.cpp` — 灯光
- `light_tree.cpp` — 灯光树
- `light_tree_debug.cpp` — 灯光树调试
- `pass.cpp` — 渲染通道

### 着色器
- `shader.cpp` — 着色器管理
- `shader_graph.cpp` — 着色器图
- `shader_nodes.cpp` — 着色器节点
- `svm.cpp` — SVM 字节码编译
- `osl.cpp` — OSL 着色器集成
- `colorspace.cpp` — 色彩空间管理
- `constant_fold.cpp` — 常量折叠优化

### 图像
- `image.cpp` — 图像管理
- `image_oiio.cpp` — OpenImageIO 图像加载
- `image_sky.cpp` — 天空模型图像
- `image_vdb.cpp` — OpenVDB 体素图像

### 基础设施
- `scene.cpp` — 场景核心
- `devicescene.cpp` — 设备场景数据
- `procedural.cpp` — 过程化几何
- `bake.cpp` — 烘焙
- `stats.cpp` — 统计
- `tables.cpp` — 查找表
- `tabulated_sobol.cpp` — Sobol 序列表

## 包含目录 (INC)

- `..` — 上级目录（cycles 根目录，用于 `scene/`、`util/` 等路径）
- `../../third_party/sky/include` — 天空模型头文件
- `../../third_party/mikktspace` — MikkTSpace 切线计算库

## 链接库 (LIB)

### 核心依赖
- `cycles_bvh` — 层次包围体(BVH)模块
- `cycles_device` — 设备抽象层
- `cycles_integrator` — 积分器模块
- `cycles_subd` — 曲面细分模块
- `cycles_util` — 工具库

### 天空模型
- `extern_sky`（独立仓库）或 `bf_intern_sky`（Blender 集成）

### 可选依赖
- **OSL**: `cycles_kernel_osl` + `${OSL_LIBRARIES}` — 当 `WITH_CYCLES_OSL` 启用
- **OpenColorIO**: `${OPENCOLORIO_LIBRARIES}` — 当 `WITH_OPENCOLORIO` 启用，添加 `-DWITH_OCIO` 定义
- **OpenVDB**: `${OPENVDB_LIBRARIES}` — 当 `WITH_OPENVDB` 启用
- **NanoVDB**: 仅包含路径 `${NANOVDB_INCLUDE_DIRS}` — 当 `WITH_NANOVDB` 启用

## 条件编译定义

- `-DWITH_OCIO` — 启用 OpenColorIO 色彩管理
- `-DOpenColorIO_SKIP_IMPORTS` — Windows 下非 USD 覆盖时跳过 OCIO 导入符号

## 依赖关系

场景模块是 Cycles 的中间层，依赖底层的 BVH、设备、积分器和工具模块，被上层的应用集成层（如 Blender 集成或独立渲染器）调用。

## 关联文件

- 所有 `src/scene/` 下的 `.cpp` 和 `.h` 文件均由此构建脚本管理
- `src/bvh/CMakeLists.txt` — BVH 模块
- `src/device/CMakeLists.txt` — 设备模块
- `src/integrator/CMakeLists.txt` — 积分器模块
- `src/subd/CMakeLists.txt` — 曲面细分模块
