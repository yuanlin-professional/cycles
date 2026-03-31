# FindWebP.cmake - WebP 查找模块

## 概述

该模块用于查找 WebP 库。WebP 是由 Google 开发的现代图像格式，提供有损和无损压缩，相比 JPEG 和 PNG 具有更好的压缩率。Cycles 渲染器使用 WebP 支持 WebP 格式纹理和图像的读取与写入。

## 查找逻辑

### 搜索路径

模块按以下优先级搜索 WebP：

1. CMake 变量 `WEBP_ROOT_DIR`（若已定义）
2. 环境变量 `WEBP_ROOT_DIR`
3. 系统路径 `/opt/lib/webp`

### 搜索过程

1. **头文件搜索**：在搜索路径的 `include` 子目录中查找 `webp/types.h`。
2. **组件库搜索**：遍历以下四个组件，在 `lib64`、`lib` 和 `lib/static` 子目录中查找对应的库文件：
   - `webp` — WebP 核心编解码库
   - `webpmux` — WebP 复用器库（用于处理动画和元数据）
   - `webpdemux` — WebP 解复用器库
   - `sharpyuv` — SharpYUV 库（WebP 1.3 新增，用于高质量 RGB 到 YUV 转换）
3. **核心库验证**：若未找到核心 `webp` 库，直接将 `WEBP_FOUND` 设为 `FALSE`，不执行后续检测。

### 版本要求

模块未指定最低版本要求。`sharpyuv` 组件为可选组件，缺失不会导致查找失败。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `WEBP_FOUND` | 布尔值，指示是否成功找到 WebP |
| `WEBP_INCLUDE_DIRS` | WebP 头文件目录列表 |
| `WEBP_LIBRARIES` | 需要链接的 WebP 库列表（包含所有找到的组件库） |
| `WEBP_LIBRARY_DIR` | WebP 核心库文件所在目录 |
| `WEBP_ROOT_DIR` | WebP 的搜索基础目录（支持环境变量） |

### 内部变量（非公开接口）

| 变量名 | 说明 |
|--------|------|
| `WEBP_WEBP_LIBRARY` | WebP 核心库文件路径 |
| `WEBP_WEBPMUX_LIBRARY` | WebP Mux 库文件路径 |
| `WEBP_WEBPDEMUX_LIBRARY` | WebP Demux 库文件路径 |
| `WEBP_SHARPYUV_LIBRARY` | SharpYUV 库文件路径 |

## 依赖关系

- 无直接 CMake 模块依赖
- 使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块
