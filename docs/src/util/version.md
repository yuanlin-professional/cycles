# version.h - Cycles 版本号定义

## 概述

`version.h` 定义了 Cycles 渲染器的版本号宏，包括主版本号、次版本号和补丁号，以及将其组合为版本字符串的辅助宏。当前版本为 5.0.0。

## 类与结构体

无类或结构体定义。

## 核心函数/宏定义

### 版本号宏

| 宏 | 值 | 说明 |
|----|----|------|
| `CYCLES_VERSION_MAJOR` | `5` | 主版本号 |
| `CYCLES_VERSION_MINOR` | `0` | 次版本号 |
| `CYCLES_VERSION_PATCH` | `0` | 补丁版本号 |

### 版本字符串宏

| 宏 | 说明 |
|----|------|
| `CYCLES_MAKE_VERSION_STRING2(a, b, c)` | 内部辅助宏，将三个参数字符串化并用 "." 连接 |
| `CYCLES_MAKE_VERSION_STRING(a, b, c)` | 对参数进行宏展开后调用 `_STRING2` |
| `CYCLES_VERSION_STRING` | 最终版本字符串，展开为 `"5.0.0"` |

**字符串化技巧**：使用两层宏（`MAKE_VERSION_STRING` -> `MAKE_VERSION_STRING2`）确保宏参数在字符串化（`#`）之前被完全展开。

## 依赖关系

- **内部头文件**: 无
- **外部依赖**: 无
- **被引用**: `app/cycles_standalone.cpp`, `app/opengl/window.cpp`

## 实现细节

1. **NOLINT 标记**：版本号定义被 `NOLINTBEGIN` / `NOLINTEND` 包裹，禁止静态分析工具对魔法数字的告警。

2. **独立于 Blender**：Cycles 维护独立的版本号，与 Blender 主版本号分离，便于作为独立渲染库使用。

3. **编译期常量**：所有版本信息在预处理阶段确定，无运行时开销。

## 关联文件

- `app/cycles_standalone.cpp` - 独立应用中显示版本信息
- `app/opengl/window.cpp` - OpenGL 窗口标题中显示版本
