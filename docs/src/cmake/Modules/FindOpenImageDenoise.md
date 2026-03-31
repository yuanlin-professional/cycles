# FindOpenImageDenoise.cmake - OpenImageDenoise 查找模块

## 概述

该模块用于查找 Intel OpenImageDenoise（OIDN）库。OpenImageDenoise 是 Intel 开发的基于深度学习的高性能图像降噪库，利用训练好的神经网络模型对蒙特卡洛光线追踪渲染的图像进行降噪处理。在 Cycles 渲染器中，OIDN 用于在渲染过程中或渲染完成后对图像进行 AI 降噪，显著减少达到清晰图像所需的采样数。

## 查找逻辑

模块按照以下优先级搜索 OpenImageDenoise：

1. **用户指定路径**：若定义了 `OPENIMAGEDENOISE_ROOT_DIR`（CMake 变量或环境变量），则优先在该路径下搜索。
2. **系统默认路径**：`/opt/lib/openimagedenoise`

### 头文件搜索

在搜索目录的 `include` 子目录下查找 `OpenImageDenoise/oidn.h`。

### 库文件搜索

模块搜索两类组件：

**主要组件**（必须找到）：
- `OpenImageDenoise`：OIDN 主库

**静态链接附加组件**（可选，用于静态构建）：
- `common`：公共基础库
- `dnnl_cpu`：Intel DNNL CPU 后端
- `dnnl_common`：Intel DNNL 公共库
- `mkldnn`：Intel MKL-DNN 库（旧版本名称）
- `dnnl`：Intel DNNL 库

> 注意：`dnnl_cpu` 在列表中出现两次，用于解决循环依赖问题。不同版本的 OIDN 所需的静态依赖库名称不同，模块会列出所有可能的名称，缺失的库会被跳过。

所有组件在搜索目录的 `lib64`、`lib` 子目录下查找。

### 版本要求

该模块未设置特定的版本要求。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `OPENIMAGEDENOISE_FOUND` | 布尔值，若为 `FALSE` 则表示未找到 OpenImageDenoise |
| `OPENIMAGEDENOISE_INCLUDE_DIRS` | OpenImageDenoise 头文件目录 |
| `OPENIMAGEDENOISE_LIBRARIES` | 链接 OpenImageDenoise 所需的全部库文件列表（包含静态依赖） |
| `OPENIMAGEDENOISE_ROOT_DIR` | 搜索 OpenImageDenoise 的根目录，可通过环境变量设置 |

### 内部变量（高级）

| 变量名 | 说明 |
|--------|------|
| `OPENIMAGEDENOISE_INCLUDE_DIR` | OpenImageDenoise 头文件目录（单数形式） |
| `OPENIMAGEDENOISE_LIBRARY` | OpenImageDenoise 主库文件路径（单数形式） |
| `OPENIMAGEDENOISE_OPENIMAGEDENOISE_LIBRARY` | OpenImageDenoise 主库路径 |
| `OPENIMAGEDENOISE_COMMON_LIBRARY` | common 静态库路径（可选） |
| `OPENIMAGEDENOISE_DNNL_CPU_LIBRARY` | dnnl_cpu 静态库路径（可选） |
| `OPENIMAGEDENOISE_DNNL_COMMON_LIBRARY` | dnnl_common 静态库路径（可选） |
| `OPENIMAGEDENOISE_MKLDNN_LIBRARY` | mkldnn 静态库路径（可选） |
| `OPENIMAGEDENOISE_DNNL_LIBRARY` | dnnl 静态库路径（可选） |

## 依赖关系

该模块使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块来处理标准的查找参数。无其他外部 Find 模块依赖。

> **注意**：静态链接时，OpenImageDenoise 依赖 Intel 的深度神经网络库（DNNL/MKL-DNN），模块会自动搜索这些附加依赖。动态链接时，只需主库 `OpenImageDenoise` 即可。
