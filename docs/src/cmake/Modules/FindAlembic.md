# FindAlembic.cmake - Alembic 查找模块

## 概述

该模块用于查找 Alembic 库。Alembic 是一个开源的计算机图形交换框架，用于在不同的数字内容创建（DCC）工具之间交换几何体和动画数据。在 Cycles 渲染器中，Alembic 用于导入和处理场景几何数据。

## 查找逻辑

模块按照以下优先级搜索 Alembic：

1. **用户指定路径**：若定义了 `ALEMBIC_ROOT_DIR`（CMake 变量或环境变量），则优先在该路径下搜索。
2. **系统默认路径**：`/opt/lib/alembic`

### 头文件搜索

在搜索目录的 `include` 子目录下查找 `Alembic/Abc/All.h`。

### 库文件搜索

在搜索目录的 `lib64`、`lib`、`lib/static` 子目录下查找名为 `Alembic` 的库文件。

### 版本要求

该模块未设置特定的版本要求。

## 输出变量

| 变量名 | 说明 |
|--------|------|
| `ALEMBIC_FOUND` | 布尔值，若为 `FALSE` 则表示未找到 Alembic |
| `ALEMBIC_INCLUDE_DIRS` | Alembic 头文件目录，在 `ALEMBIC_INCLUDE_DIR` 被找到时设置 |
| `ALEMBIC_LIBRARIES` | 链接 Alembic 所需的库文件列表 |
| `ALEMBIC_ROOT_DIR` | 搜索 Alembic 的根目录，可通过环境变量设置 |

### 内部变量（高级）

| 变量名 | 说明 |
|--------|------|
| `ALEMBIC_INCLUDE_DIR` | Alembic 头文件目录（单数形式） |
| `ALEMBIC_LIBRARY` | Alembic 库文件路径（单数形式） |

## 依赖关系

该模块使用 CMake 内置的 `FindPackageHandleStandardArgs` 模块来处理标准的查找参数。无其他外部 Find 模块依赖。
