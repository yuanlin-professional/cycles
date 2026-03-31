# CMakeLists.txt - subd 模块构建配置

## 概述

本文件是 Cycles 渲染器细分曲面（subd）模块的 CMake 构建配置文件。它定义了该模块的源文件列表和头文件列表，并通过 `cycles_add_library` 宏将其编译为 `cycles_subd` 静态库。该模块是 Cycles 细分曲面处理管线的核心组件，包含面片分割、切割、属性插值和 OpenSubdiv 集成等功能。

## 构建选项 / 变量

### INC
- **值**: `..`（指向 `src/` 目录）
- **说明**: 内部头文件包含路径，允许使用 `subd/xxx.h`、`scene/xxx.h`、`util/xxx.h` 等相对路径引用头文件

### INC_SYS
- **值**: 空
- **说明**: 系统/外部头文件包含路径（当前未设置额外路径；OpenSubdiv 头文件路径由上层 CMake 配置提供）

### SRC
- **说明**: 编译的源文件列表
  - `dice.cpp` — 边缘切割（EdgeDice）实现
  - `interpolation.cpp` — 属性插值实现
  - `osd.cpp` — OpenSubdiv 集成封装
  - `patch.cpp` — 面片求值实现
  - `split.cpp` — DiagSplit 分割算法实现

### SRC_HEADERS
- **说明**: 头文件列表（不参与编译，但被 IDE 索引和安装）
  - `dice.h` — EdgeDice 和 SubdParams 声明
  - `interpolation.h` — SubdAttributeInterpolation 声明
  - `osd.h` — OpenSubdiv 封装声明（条件编译）
  - `patch.h` — Patch 基类及 LinearQuadPatch / BicubicPatch 声明
  - `split.h` — DiagSplit 声明
  - `subpatch.h` — SubPatch / SubEdge 数据结构定义

### LIB
- **值**: 空
- **说明**: 额外链接库（当前为空；OpenSubdiv 链接由上层配置处理）

### 构建目标
- **库名**: `cycles_subd`
- **构建方式**: 通过 `cycles_add_library` 宏创建（Cycles 自定义宏，封装了 `add_library` 及相关配置）

## 依赖关系

- **上游依赖**: `cycles_util`（工具库），`cycles_scene`（场景库，编译时头文件依赖）
- **外部依赖**: OpenSubdiv（可选，通过 `WITH_OPENSUBDIV` 宏控制）
- **下游依赖**: `cycles_scene`（链接时依赖 cycles_subd）
