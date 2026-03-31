# macros.cmake - Cycles 构建系统核心宏与函数

## 概述

该文件定义了 Cycles 构建系统中的核心宏和函数，包括库目标创建、外部库依赖链接、共享库安装以及库检测警告等功能。它是 Cycles 各子模块 `CMakeLists.txt` 中最常被调用的工具文件。

## 构建选项 / 变量

该文件不定义缓存变量，但依赖以下外部变量：

| 变量名 | 说明 |
|--------|------|
| `IDE_GROUP_PROJECTS_IN_FOLDERS` | 是否在 IDE 中按目录分组项目 |
| `WITH_CYCLES_OSL` | OSL 开关 |
| `WITH_CYCLES_EMBREE` | Embree 开关 |
| `EMBREE_SYCL_SUPPORT` | Embree 是否启用 SYCL 支持 |
| `WITH_OPENSUBDIV` | OpenSubdiv 开关 |
| `WITH_OPENCOLORIO` | OpenColorIO 开关 |
| `WITH_OPENVDB` | OpenVDB 开关 |
| `WITH_OPENIMAGEDENOISE` | OpenImageDenoise 开关 |
| `WITH_PATH_GUIDING` | 路径引导开关 |
| `WITH_IMAGE_WEBP` | WebP 图像支持开关 |
| `WITH_USD` | USD 开关 |
| `WITH_CYCLES_DEVICE_CUDA` | CUDA 设备开关 |
| `WITH_CYCLES_DEVICE_OPTIX` | OptiX 设备开关 |
| `WITH_CYCLES_DEVICE_HIP` | HIP 设备开关 |
| `WITH_CUDA_DYNLOAD` | CUDA 动态加载开关 |
| `WITH_HIP_DYNLOAD` | HIP 动态加载开关 |
| `WITH_STRICT_BUILD_OPTIONS` | 严格构建选项模式 |
| `CYCLES_STANDALONE_REPOSITORY` | 是否为独立仓库构建 |
| `PLATFORM_BUNDLED_LIBRARIES_RELEASE` | Release 构建捆绑库列表 |
| `PLATFORM_BUNDLED_LIBRARIES_DEBUG` | Debug 构建捆绑库列表 |
| `PLATFORM_LIB_INSTALL_DIR` | 库安装目录 |
| `PLATFORM_LINKLIBS` | 平台链接库列表 |

## 关键逻辑

### 函数：`cycles_set_solution_folder`

为 IDE（如 Visual Studio）设置项目的解决方案文件夹：

- 仅在 `IDE_GROUP_PROJECTS_IN_FOLDERS` 开启时生效
- 根据源文件目录结构自动计算文件夹路径
- 通过 `FOLDER` 属性实现 IDE 中的项目分组

### 宏：`cycles_add_library`

Cycles 专用的库创建宏，替代原生 `add_library`：

- 创建库目标并处理依赖链接
- 核心功能：正确处理 MSVC 的 `optimized`/`debug` 库变体链接
- 通过状态机遍历 `library_deps` 列表，识别 `optimized`、`debug` 关键字并相应调用 `target_link_libraries`
- 自动调用 `cycles_set_solution_folder` 设置 IDE 项目分组

**MSVC 库变体链接说明**：

CMake 的库变量可能包含构建类型前缀，例如：
```
optimized libfoo.lib debug libfoo_d.lib
```
当这些值通过宏参数传递时会被展平为列表。该宏通过状态机正确解析这种格式，将每个库分配到对应的构建类型。

### 宏：`cycles_external_libraries_append`

将所有必要的外部库追加到指定的库列表中：

**平台框架**：
- macOS：`Foundation`（必需），`CoreVideo`、`Cocoa`、`OpenGL`（USD 时），`IOKit`、`Carbon`（OCIO 时），`Accelerate`（ARM64 OIDN 时）
- Windows：`opengl32`（USD 时）
- Linux/Unix：`-lm -lc -lutil`

**可选库（根据功能开关）**：
- OSL、Embree（含可选 SYCL）、OpenSubdiv、OpenColorIO
- OpenVDB（含可选 Blosc）、OpenImageDenoise、OpenPGL、WebP

**必需库**：
- OpenImageIO、PNG、JPEG、TIFF、OpenJPEG、OpenEXR、PugiXML
- Python、Zlib、PThreads

**GPU 设备库**：
- CUDA/OptiX：使用 `extern_cuew`（动态加载）或 `CUDA_CUDA_LIBRARY`
- HIP：使用 `extern_hipew`（动态加载）

**兼容性库**：
- Linux 独立构建：`extern_libc_compat`
- Blender 集成构建：`bf_intern_libc_compat`、`bf_intern_guardedalloc`

### 宏：`cycles_install_libraries`

安装捆绑的共享库：

- Release/RelWithDebInfo/MinSizeRel 配置安装 Release 版本库
- Debug 配置安装 Debug 版本库
- 安装路径为 `target_dir + PLATFORM_LIB_INSTALL_DIR`

### 宏：`set_and_warn_library_found`

库检测结果的统一处理宏：

- 参数：库名称、检测结果变量、功能开关变量
- 如果库未找到但功能已开启：
  - 严格模式（`WITH_STRICT_BUILD_OPTIONS`）：报错终止构建
  - 非严格模式：输出警告并自动关闭对应功能开关

## 依赖关系

- 无直接 include 的其他 cmake 文件
- 依赖 `external_libs.cmake` 中设置的各库变量
- 依赖 `configure_build.cmake` 中设置的平台变量
