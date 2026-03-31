# FindEmbree.cmake - Embree 查找模块

## 概述

该模块用于查找 Intel Embree 光线追踪库。Embree 是 Intel 开发的高性能光线追踪内核库，提供优化的光线-几何体相交计算。在 Cycles 渲染器中，Embree 是核心的光线追踪加速后端，用于 BVH 加速结构构建和光线遍历。

## 查找逻辑

模块按照以下优先级搜索 Embree：

1. **用户指定路径**：若定义了 `EMBREE_ROOT_DIR`（CMake 变量或环境变量），则优先在该路径下搜索。
2. **系统默认路径**：`/opt/lib/embree`

### 头文件搜索

在搜索目录的 `include` 子目录下查找 `embree4/rtcore.h` 或 `embree3/rtcore.h`。模块通过检查 `embree4/rtcore_config.h` 是否存在来确定主版本号（Embree 3 或 Embree 4）。

### 版本检测与静态库支持

模块读取 `rtcore_config.h` 配置头文件，检测以下特性：

- **静态库标志**：检查 `EMBREE_STATIC_LIB` 宏定义
- **SYCL 支持**：检查 `EMBREE_SYCL_SUPPORT` 宏定义

### 库文件搜索

根据静态库/动态库模式，搜索的组件有所不同：

**动态库模式**：
- `embree3` 或 `embree4`（根据版本）
- 若支持 SYCL：`embree4_sycl`

**静态库模式**（x86_64 架构）：
- `embree3` 或 `embree4`
- SIMD 组件：`embree_sse42`、`embree_avx`、`embree_avx2`
- GPU 组件（若支持 SYCL）：`embree4_sycl`、`embree_rthwif`
- 内部组件：`lexers`、`math`、`simd`、`sys`、`tasking`

所有组件在搜索目录的 `lib64`、`lib` 子目录下查找。

### 版本要求

支持 Embree 3 和 Embree 4，自动检测主版本号。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `EMBREE_FOUND` | 布尔值，若为 `FALSE` 则表示未找到 Embree |
| `EMBREE_INCLUDE_DIRS` | Embree 头文件目录 |
| `EMBREE_LIBRARIES` | 链接 Embree 所需的全部库文件列表 |
| `EMBREE_ROOT_DIR` | 搜索 Embree 的根目录，可通过环境变量设置 |
| `EMBREE_MAJOR_VERSION` | Embree 主版本号（3 或 4） |
| `EMBREE_STATIC_LIB` | 布尔值，是否为静态库构建 |
| `EMBREE_SYCL_SUPPORT` | 布尔值，是否支持 SYCL（Intel GPU） |

### 内部变量（高级）

| 变量名 | 说明 |
|--------|------|
| `EMBREE_INCLUDE_DIR` | Embree 头文件目录（单数形式） |
| `EMBREE_${UPPERCOMPONENT}_LIBRARY` | 各 Embree 组件库的路径 |

## 依赖关系

该模块使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块。无其他外部 Find 模块依赖。静态库构建时，需要架构相关的 SIMD 组件库。
