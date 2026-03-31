# cmake - 构建系统

## 概述

`src/cmake/` 目录包含 Cycles 路径追踪渲染引擎的 CMake 构建系统配置。该构建系统负责平台检测、编译器配置、外部依赖库查找与链接，以及构建目标的定义。

Cycles 支持以下平台和编译器：

- **Windows**：MSVC（Visual Studio 2019+），支持 x64 和 ARM64 架构
- **macOS**：Clang/Xcode，支持 x86_64 和 arm64 架构（最低部署目标 macOS 11.2）
- **Linux**：GCC 11.2+/Clang，支持 x86_64 和 aarch64 架构

所有平台均要求 C++17 标准。CMake 最低版本要求为 3.10。

构建系统支持独立编译（Standalone）模式，也可作为 Blender 源码树的一部分进行编译。当独立编译时，会使用预编译的第三方库（通过 Git 子模块管理）或系统库。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `configure_build.cmake` | CMake 脚本 | 全局构建配置：输出目录设置、C++17 标准启用、各编译器（GCC/Clang/MSVC）的编译标志和链接器选项 |
| `detect_platform.cmake` | CMake 脚本 | macOS 平台检测：自动识别 CPU 架构（x86_64/arm64）并设置 `CMAKE_OSX_ARCHITECTURES` 和最低部署目标 |
| `compiler_utils.cmake` | CMake 脚本 | 编译器工具函数：检查编译器标志支持（`ADD_CHECK_CXX_COMPILER_FLAGS`）、延迟安装机制（`delayed_install`/`delayed_do_install`）、移除编译标志（`remove_cc_flag`） |
| `macros.cmake` | CMake 脚本 | Cycles 专用宏定义：库添加（`cycles_add_library`）、外部库追加（`cycles_external_libraries_append`）、共享库安装（`cycles_install_libraries`）、IDE 文件夹组织（`cycles_set_solution_folder`） |
| `external_libs.cmake` | CMake 脚本 | 外部依赖库检测与配置的核心文件：预编译库路径设置、所有第三方库的查找与链接配置、平台共享库打包 |
| `make_update.py` | Python 脚本 | 更新工具：拉取 Cycles Git 仓库最新代码，并更新预编译库子模块（支持 `--no-libraries`、`--no-cycles`、`--legacy` 等参数） |
| `make_format.py` | Python 脚本 | 代码格式化工具：使用 clang-format 对源码文件（.c/.cpp/.h/.osl/.glsl/.cu/.cl）进行批量格式化，支持多进程并行处理 |
| `make_utils.py` | Python 脚本 | 通用工具函数库：为 `make_update.py` 等脚本提供 Git 操作封装（子模块管理、分支检测、LFS 支持）和进程调用工具 |
| `msvc_arch_flags.c` | C 源文件 | MSVC 架构标志检测程序：在运行时检测 CPU 支持的指令集（AVX2/AVX），输出相应的 `/arch:` 编译标志 |
| `zstd_compress.cpp` | C++ 源文件 | Zstandard 压缩工具：独立的文件压缩程序，使用 ZSTD 库以压缩级别 19 对文件进行压缩（用于内核二进制文件的压缩） |
| `Modules/` | 目录 | Find 模块集合：包含 35 个 `Find*.cmake` 模块，用于定位各外部依赖库的头文件和库文件 |

## 构建选项

以下是在根 `CMakeLists.txt` 中定义的主要 CMake 构建选项：

### 通用选项

| 选项 | 默认值 | 说明 |
|------|--------|------|
| `WITH_LIBS_PRECOMPILED` | `ON` | 使用预编译的第三方库依赖 |
| `WITH_STRICT_BUILD_OPTIONS` | `OFF` | 当构建选项的依赖不满足时报错而非自动禁用 |
| `WITH_LEGACY_LIBRARIES` | `OFF` | 使用旧版 VFX Platform 2024 库，以兼容 Houdini 或 USD |

### 库依赖选项

| 选项 | 默认值 | 说明 |
|------|--------|------|
| `WITH_CYCLES_ALEMBIC` | `ON` | 构建 Alembic 程序化几何支持 |
| `WITH_CYCLES_EMBREE` | `ON` | 构建 Intel Embree 光线追踪库支持 |
| `WITH_CYCLES_LOGGING` | `OFF` | 构建日志支持（使用 glog） |
| `WITH_CYCLES_OPENCOLORIO` | `ON` | 构建 OpenColorIO 颜色管理支持 |
| `WITH_CYCLES_OPENIMAGEDENOISE` | `ON` | 构建 Intel OpenImageDenoise 降噪支持 |
| `WITH_CYCLES_OPENSUBDIV` | `ON` | 构建 OpenSubdiv 细分曲面支持 |
| `WITH_CYCLES_OPENVDB` | `ON` | 构建 OpenVDB 稀疏体积数据支持 |
| `WITH_CYCLES_NANOVDB` | `ON` | 构建 NanoVDB 支持（GPU 体积渲染） |
| `WITH_CYCLES_OSL` | `ON` | 构建 Open Shading Language（着色器语言）支持 |
| `WITH_CYCLES_USD` | `ON` | 构建 Universal Scene Description 支持 |
| `WITH_CYCLES_PUGIXML` | `ON` | 构建 XML 解析支持 |

### GPU 设备选项

| 选项 | 默认值 | 说明 |
|------|--------|------|
| `WITH_CYCLES_DEVICE_CUDA` | `ON` | 启用 NVIDIA CUDA 计算设备支持（macOS 和 Windows ARM64 上不可用） |
| `WITH_CYCLES_DEVICE_OPTIX` | `ON` | 启用 NVIDIA OptiX 光线追踪设备支持 |
| `WITH_CYCLES_CUDA_BINARIES` | `OFF` | 预编译 CUDA 内核二进制文件 |
| `WITH_CUDA_DYNLOAD` | `ON` | 在运行时动态加载 CUDA 库 |
| `WITH_CYCLES_DEVICE_HIP` | `ON` | 启用 AMD HIP 计算设备支持 |
| `WITH_CYCLES_HIP_BINARIES` | `OFF` | 预编译 HIP 内核二进制文件 |
| `WITH_CYCLES_DEVICE_HIPRT` | `OFF` | 启用 AMD HIP RT 光线追踪支持 |
| `WITH_CYCLES_DEVICE_METAL` | `ON` | 启用 Apple Metal 计算设备支持（仅 macOS） |
| `WITH_CYCLES_DEVICE_ONEAPI` | `OFF` | 启用 Intel oneAPI 计算设备支持 |
| `WITH_CYCLES_ONEAPI_BINARIES` | `OFF` | 预编译 oneAPI AOT 内核二进制文件 |

### 开发调试选项

| 选项 | 默认值 | 说明 |
|------|--------|------|
| `WITH_CYCLES_DEBUG` | `OFF` | 启用调试选项（如 MIS 调试） |
| `WITH_CYCLES_DEBUG_NAN` | `OFF` | 启用 NaN 和无效值检测断言 |
| `WITH_CYCLES_NATIVE_ONLY` | `OFF` | 仅构建适配当前 CPU 的内核（开发专用） |
| `WITH_CYCLES_STANDALONE_GUI` | `OFF` | 构建带 GUI 的独立可执行程序（需要 SDL2） |

### Hydra 渲染委托

| 选项 | 默认值 | 说明 |
|------|--------|------|
| `WITH_CYCLES_HYDRA_RENDER_DELEGATE` | `ON` | 构建 Cycles Hydra 渲染委托（需要 USD） |

## Find 模块

`Modules/` 目录下包含以下 Find 模块，用于定位第三方依赖库：

### 渲染与场景

| 模块 | 说明 |
|------|------|
| `FindEmbree.cmake` | 查找 Intel Embree 光线追踪库（支持 v3/v4，检测静态库与 SYCL 支持） |
| `FindOptiX.cmake` | 查找 NVIDIA OptiX 光线追踪 SDK（仅头文件库，检测版本号） |
| `FindOSL.cmake` | 查找 Open Shading Language（着色器编译器与运行时库） |
| `FindOpenSubdiv.cmake` | 查找 Pixar OpenSubdiv 细分曲面库 |
| `FindAlembic.cmake` | 查找 Alembic 场景交换格式库 |
| `FindUSD.cmake` | 查找 Universal Scene Description 库 |
| `FindUSDPixar.cmake` | 查找 Pixar 官方 USD 发行版 |
| `FindUSDHoudini.cmake` | 查找 SideFX Houdini 捆绑的 USD 库 |

### 图像与颜色

| 模块 | 说明 |
|------|------|
| `FindOpenImageIO.cmake` | 查找 OpenImageIO 图像读写库 |
| `FindOpenEXR.cmake` | 查找 OpenEXR 高动态范围图像库（含 Imath） |
| `FindOpenColorIO.cmake` | 查找 OpenColorIO 颜色管理库 |
| `FindOpenImageDenoise.cmake` | 查找 Intel Open Image Denoise 人工智能降噪库 |
| `FindOpenJPEG.cmake` | 查找 OpenJPEG 图像编解码库 |
| `FindWebP.cmake` | 查找 WebP 图像格式库 |

### 体积数据

| 模块 | 说明 |
|------|------|
| `FindOpenVDB.cmake` | 查找 OpenVDB 稀疏体积数据库 |
| `FindNanoVDB.cmake` | 查找 NanoVDB（OpenVDB 的轻量级 GPU 版本，仅头文件） |
| `FindBlosc.cmake` | 查找 Blosc 压缩库（OpenVDB 依赖） |

### GPU 计算与并行

| 模块 | 说明 |
|------|------|
| `FindHIP.cmake` | 查找 AMD HIP 编译器和运行时 |
| `FindHIPRT.cmake` | 查找 AMD HIP RT 光线追踪库 |
| `FindSYCL.cmake` | 查找 Intel oneAPI DPC++ SYCL 编译器和运行时 |
| `FindLevelZero.cmake` | 查找 Intel Level Zero GPU 驱动 API |

### 编译器工具链

| 模块 | 说明 |
|------|------|
| `FindLLVM.cmake` | 查找 LLVM 编译器基础设施 |
| `FindClang.cmake` | 查找 Clang 编译器前端（OSL 依赖） |

### 系统与工具库

| 模块 | 说明 |
|------|------|
| `FindTBB.cmake` | 查找 Intel Threading Building Blocks（oneTBB）并行库 |
| `FindPugiXML.cmake` | 查找 pugixml XML 解析库 |
| `FindZstd.cmake` | 查找 Zstandard 压缩库 |
| `FindGlog.cmake` | 查找 Google 日志库 |
| `FindGflags.cmake` | 查找 Google 命令行标志解析库 |
| `FindSDL2.cmake` | 查找 SDL2 多媒体库（独立 GUI 模式使用） |
| `FindPythonLibsUnix.cmake` | 在 Unix 系统上查找 Python 库（USD 依赖） |
| `Findsse2neon.cmake` | 查找 sse2neon 头文件库（在 ARM 上将 SSE 内联函数转换为 NEON） |

### 图形显示

| 模块 | 说明 |
|------|------|
| `FindEpoxy.cmake` | 查找 libepoxy OpenGL 函数加载库 |
| `FindLibEpoxy.cmake` | 查找 libepoxy 的备选查找模块（pkg-config 方式） |
| `FindGLEW.cmake` | 查找 GLEW OpenGL 扩展加载库 |

## 平台检测与编译器配置

### 平台检测流程

1. **`detect_platform.cmake`**：在 macOS 上执行 `uname -m` 检测 CPU 架构，自动设置 `CMAKE_OSX_ARCHITECTURES`（x86_64 或 arm64），并设定最低部署目标为 macOS 11.2。

2. **`external_libs.cmake`**：根据操作系统和 `CMAKE_SYSTEM_PROCESSOR` 确定预编译库平台标识符：
   - Windows: `windows_x64` / `windows_arm64`
   - macOS: `macos_x64` / `macos_arm64`
   - Linux: `linux_x64` / `linux_arm64`

### 编译器配置（`configure_build.cmake`）

| 编译器 | 主要标志 |
|--------|---------|
| GCC | `-Wall -Wno-sign-compare -fno-strict-aliasing -fPIC -std=c++17` |
| Clang | `-Wall -Wno-sign-compare -fno-strict-aliasing -fPIC -std=c++17`（macOS 额外 `-stdlib=libc++`） |
| MSVC | `/nologo /J /Gd /EHsc /bigobj /MP /std:c++17 /Zc:__cplusplus /Zc:inline-`（链接 `psapi Version Dbghelp Shlwapi`） |

### 预编译库机制

当 `WITH_LIBS_PRECOMPILED=ON` 且 `lib/<platform>` 目录存在时，构建系统将：
- 设置所有第三方库的 `ROOT_DIR` 指向预编译库目录
- 将预编译库路径加入 `CMAKE_PREFIX_PATH`
- 屏蔽系统库路径（`CMAKE_IGNORE_PATH`）以避免冲突
- 收集平台共享库（DLL/dylib/so）用于安装打包

当预编译库不存在时，回退到使用 `find_package()` 查找系统安装的库。

## 依赖关系

### 上游依赖（本模块依赖）

- **CMake 内置模块**：`CheckCXXCompilerFlag`、`CheckCXXSourceCompiles`、`FindPackageHandleStandardArgs`、`InstallRequiredSystemLibraries`、`CTest`
- **Git**：`make_update.py` 和 `make_utils.py` 依赖 Git 进行仓库管理和预编译库子模块更新
- **Git LFS**：预编译库通过 Git LFS 存储大文件
- **Python 3**：运行 `make_update.py`、`make_format.py` 等构建辅助脚本
- **clang-format**：`make_format.py` 依赖 clang-format 进行代码格式化

### 下游依赖（依赖本模块）

- **`CMakeLists.txt`（根目录）**：通过 `include()` 加载 `compiler_utils`、`macros`、`detect_platform`、`external_libs`、`configure_build` 五个核心脚本
- **`src/` 下所有子目录的 `CMakeLists.txt`**：使用 `macros.cmake` 中定义的 `cycles_add_library`、`cycles_external_libraries_append` 等宏来构建各模块
- **所有 Cycles 构建目标**：依赖 `external_libs.cmake` 中检测到的第三方库变量（如 `EMBREE_LIBRARIES`、`OPENEXR_LIBRARIES` 等）
- **安装目标**：依赖 `compiler_utils.cmake` 中的 `delayed_install` / `delayed_do_install` 机制和 `macros.cmake` 中的 `cycles_install_libraries`

## 参见

- `CMakeLists.txt`（根目录）- 项目顶层构建入口，定义所有构建选项并引入本目录下的脚本
- `src/` - 源码目录，各子目录的 `CMakeLists.txt` 使用本目录提供的宏和变量
- `third_party/` - 第三方库源码（cuew、hipew 等动态加载封装）
- `lib/` - 预编译库子模块目录（按平台组织）
