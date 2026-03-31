# FindHIPRT.cmake - HIPRT 查找模块

## 概述

该模块用于查找 HIPRT（HIP Ray Tracing）SDK。HIPRT 是 AMD 提供的光线追踪 SDK，为 HIP 应用程序提供硬件加速的光线追踪功能。在 Cycles 渲染器中，HIPRT 用于在 AMD GPU 上实现硬件加速的光线-几何体相交计算。

## 查找逻辑

模块按照以下优先级搜索 HIPRT：

1. **用户指定路径**：若定义了 `HIPRT_ROOT_DIR`（CMake 变量），则优先在该路径下搜索。
2. **环境变量**：
   - `HIPRT_ROOT_DIR` 环境变量
   - `HIP_PATH` 环境变量（AMD SDK 内置环境变量）
3. **系统默认路径**：`/opt/lib/hiprt`

### 头文件搜索

在搜索目录的 `include` 子目录下查找 `hiprt/hiprt.h`。

### 版本检测

若找到头文件，模块会从 `hiprt/hiprt.h` 中提取 `HIPRT_VERSION_STR` 宏定义来获取版本号。

### 版本要求

该模块未设置特定的最低版本要求。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `HIPRT_FOUND` | 布尔值，若为 `FALSE` 则表示未找到 HIPRT SDK |
| `HIPRT_INCLUDE_DIR` | HIPRT 头文件目录 |
| `HIPRT_VERSION` | HIPRT 版本字符串 |

> **注意**：该模块为纯头文件 SDK 查找模块，不搜索库文件。

### 内部变量（高级）

| 变量名 | 说明 |
|--------|------|
| `HIPRT_INCLUDE_DIR` | HIPRT 头文件目录 |

## 依赖关系

该模块使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块来处理标准的查找参数。

> **相关模块**：该模块通常与 `FindHIP.cmake` 配合使用。`FindHIP.cmake` 查找 HIP 编译器工具链，而本模块查找光线追踪 SDK。
