# external_libs.cmake - 外部依赖库检测与配置

## 概述

该文件是 Cycles 构建系统中最大、最核心的配置文件之一。它负责检测和配置所有外部第三方依赖库，包括预编译库路径设置、各库的 find_package 调用、MSVC 平台的手动库路径配置，以及 GPU 计算设备（CUDA、HIP、oneAPI、Metal）的 SDK 检测。该文件同时管理捆绑共享库的收集与安装路径配置。

## 构建选项 / 变量

### 预编译库相关

| 变量名 | 说明 |
|--------|------|
| `WITH_LIBS_PRECOMPILED` | 是否使用预编译库 |
| `WITH_LEGACY_LIBRARIES` | 是否使用旧版库（兼容不支持 oneTBB 的 Houdini/USD） |
| `_cycles_lib_dir` | 预编译库根目录路径，根据平台自动确定 |
| `_cycles_lib_platform` | 平台标识字符串（如 `windows_x64`、`macos_arm64`、`linux_x64`） |

### 功能开关（控制哪些库被检测）

| 变量名 | 说明 |
|--------|------|
| `WITH_CYCLES_HYDRA_RENDER_DELEGATE` | 启用 Hydra 渲染代理 |
| `WITH_CYCLES_USD` | 启用 USD 支持 |
| `WITH_USD` | USD 总开关（由上两者触发） |
| `WITH_CYCLES_OSL` | 启用 Open Shading Language |
| `WITH_CYCLES_PATH_GUIDING` | 启用路径引导（OpenPGL） |
| `WITH_CYCLES_OPENCOLORIO` | 启用 OpenColorIO 色彩管理 |
| `WITH_CYCLES_EMBREE` | 启用 Embree 光线追踪 |
| `WITH_CYCLES_LOGGING` | 启用日志功能（Glog/Gflags） |
| `WITH_CYCLES_OPENSUBDIV` | 启用 OpenSubdiv 细分曲面 |
| `WITH_CYCLES_OPENVDB` | 启用 OpenVDB 体积数据 |
| `WITH_CYCLES_NANOVDB` | 启用 NanoVDB |
| `WITH_CYCLES_OPENIMAGEDENOISE` | 启用 OpenImageDenoise 降噪 |
| `WITH_CYCLES_ALEMBIC` | 启用 Alembic 场景交换格式 |
| `WITH_CYCLES_STANDALONE` | 独立版 Cycles |
| `WITH_CYCLES_STANDALONE_GUI` | 独立版 GUI（需要 SDL2 和 Epoxy） |
| `WITH_CYCLES_DEVICE_CUDA` | 启用 CUDA 设备 |
| `WITH_CYCLES_DEVICE_OPTIX` | 启用 OptiX 设备 |
| `WITH_CYCLES_DEVICE_HIP` | 启用 HIP 设备 |
| `WITH_CYCLES_DEVICE_HIPRT` | 启用 HIP RT |
| `WITH_CYCLES_DEVICE_ONEAPI` | 启用 oneAPI 设备 |
| `WITH_CYCLES_DEVICE_METAL` | 启用 Metal 设备（仅 macOS） |
| `WITH_CYCLES_CUDA_BINARIES` | 编译 CUDA 内核二进制 |
| `WITH_CYCLES_HIP_BINARIES` | 编译 HIP 内核二进制 |
| `WITH_CYCLES_ONEAPI_BINARIES` | 编译 oneAPI 内核二进制 |

### 各库路径变量

以下为预编译库模式下的默认搜索路径：

| 变量名 | 默认路径（相对于 `_cycles_lib_dir`） |
|--------|--------------------------------------|
| `ALEMBIC_ROOT_DIR` | `alembic` |
| `Boost_ROOT` | `boost` |
| `EMBREE_ROOT_DIR` | `embree` |
| `EPOXY_ROOT_DIR` | `epoxy` |
| `IMATH_ROOT_DIR` | `imath` |
| `GLEW_ROOT_DIR` | `glew` |
| `JPEG_ROOT` | `jpeg` |
| `MATERIALX_ROOT_DIR` | `MaterialX`（Windows）/ `materialx`（其他） |
| `NANOVDB_ROOT_DIR` | `openvdb` |
| `OPENCOLORIO_ROOT_DIR` | `opencolorio` |
| `OPENEXR_ROOT_DIR` | `openexr` |
| `OPENIMAGEDENOISE_ROOT_DIR` | `openimagedenoise` |
| `OPENIMAGEIO_ROOT_DIR` | `openimageio` |
| `OPENJPEG_ROOT_DIR` | `openjpeg` |
| `OPENSUBDIV_ROOT_DIR` | `opensubdiv` |
| `OPENVDB_ROOT_DIR` | `openvdb` |
| `OSL_ROOT_DIR` | `osl` |
| `PNG_ROOT` | `png` |
| `PUGIXML_ROOT_DIR` | `pugixml` |
| `PYTHON_ROOT_DIR` | `python` |
| `SSE2NEON_ROOT_DIR` | `sse2neon` |
| `TBB_ROOT_DIR` | `tbb` |
| `TIFF_ROOT` | `tiff` |
| `USD_ROOT_DIR` | `usd` |
| `WEBP_ROOT_DIR` | `webp` |
| `ZLIB_ROOT` | `zlib` |
| `ZSTD_ROOT_DIR` | `zstd` |
| `LEVEL_ZERO_ROOT_DIR` | `level_zero`（Windows）/ `level-zero`（其他） |
| `SYCL_ROOT_DIR` | `dpcpp` |

## 关键逻辑

### 平台检测与预编译库路径

根据操作系统和 CPU 架构确定预编译库目录：

- **macOS**：`macos_x64` 或 `macos_arm64`
- **Windows**：`windows_x64` 或 `windows_arm64`
- **Linux**：`linux_x64` 或 `linux_arm64`（要求 GCC >= 11.2）

支持旧版库兼容模式，路径格式为 `lib/legacy/{platform}`。

### USD 检测（优先执行）

USD 检测最先运行，因为它影响 C++ ABI 和库的选择：

- 支持 Houdini 环境（`HOUDINI_ROOT`）和 Pixar 独立 USD（`PXR_ROOT`）
- Windows 下手动配置 Python 3.11 路径
- 非 Windows 平台使用 `PythonLibsUnix`

### MSVC 特殊处理模式

在 MSVC + 预编译库环境下，大多数库采用手动路径设置而非 `find_package`，并区分 Release/Debug 变体：

- Release 库使用 `optimized` 关键字
- Debug 库使用 `debug` 关键字（通常以 `_d` 后缀命名）

### 宏：`_set_default`

辅助宏，仅在变量未设置时赋予默认值，避免覆盖用户自定义配置。

### 宏：`add_bundled_libraries`

收集预编译共享库以供安装：

- Windows：收集 `.dll` 文件，按 `_d.dll`/`_debug.dll` 后缀区分 Debug/Release
- macOS：收集 `.dylib*` 文件
- Linux：收集 `.so*` 文件

### GPU 设备 SDK 检测

#### CUDA
- 使用 `find_package(CUDA)` 自动定位 CUDA 工具链
- 找不到时回退到动态加载模式（`WITH_CUDA_DYNLOAD`）

#### HIP
- 要求 HIP >= 6.0.0
- 可选启用 HIP RT（光线追踪加速）
- 默认启用动态加载（`WITH_HIP_DYNLOAD`）

#### oneAPI
- 检测 SYCL 编译器和 Level Zero 运行时
- 要求 SYCL >= 6.0
- 管理 SYCL 运行时 DLL/SO 的捆绑（区分旧版 sycl7/pi_* 和新版 sycl8/ur_*）
- 检测 ocloc 离线编译器用于内核预编译

#### Metal
- 仅 macOS，检测 Metal 框架
- 要求 SDK >= 12.0（通过检测 `MTLFunctionStitching.h` 头文件）

### SSE2NEON 检测

- 通过编译测试检测 ARM Neon 支持
- 在 ARM 平台上提供 SSE 到 Neon 的指令转换

### 捆绑共享库安装路径

| 平台 | 安装路径 | 环境变量 |
|------|----------|----------|
| Windows | `.`（当前目录） | `PATH` |
| macOS | `lib` | `DYLD_LIBRARY_PATH` |
| Linux | `lib` | `LD_LIBRARY_PATH` |

Windows 平台额外安装 UCRT（通用C运行时）库。

## 依赖关系

### 内部依赖

使用了 `set_and_warn_library_found` 宏（定义在 `macros.cmake` 中）

### CMake 模块依赖

- `CheckCXXSourceCompiles`
- `InstallRequiredSystemLibraries`（仅 Windows）

### Find 模块依赖（位于 `Modules/` 目录）

- `FindUSDHoudini`、`FindUSDPixar`、`FindUSD`
- `FindOpenImageIO`、`FindPugiXML`、`FindOpenJPEG`
- `FindOpenEXR`、`FindOSL`
- `FindOpenColorIO`、`FindOpenSubdiv`、`FindOpenVDB`、`FindNanoVDB`
- `FindOpenImageDenoise`
- `FindEmbree`、`FindGlog`、`FindGflags`
- `FindTBB`、`FindEpoxy`、`FindAlembic`
- `FindSYCL`、`FindLevelZero`
- `FindHIP`、`FindHIPRT`
- `FindZstd`、`Findsse2neon`
- `FindPythonLibsUnix`、`FindMaterialX`
- `FindSDL2`
- 标准 CMake 模块：`FindZLIB`、`FindThreads`、`FindJPEG`、`FindTIFF`、`FindPNG`、`FindWebP`、`FindCUDA`
