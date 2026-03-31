# shader.h / shader.cpp - OpenGL 显示着色器封装

## 概述

本文件封装了用于渲染结果显示的 OpenGL 着色器程序 `OpenGLShader`。该着色器负责将包含渲染结果的 2D 纹理绘制到屏幕四边形上，并在片段着色器中执行硬编码的 Rec.709 gamma 校正。它属于独立应用的 OpenGL 显示模块，被 `OpenGLDisplayDriver` 作为成员对象使用，承担着将 HDR 半精度浮点像素数据转换为可显示 sRGB 输出的关键角色。

## 类与结构体

### OpenGLShader

- **继承**: 无（独立类，虚析构函数支持多态扩展）
- **功能**: 管理 OpenGL 着色器程序（顶点着色器 + 片段着色器）的编译、链接、绑定和销毁，为 `OpenGLDisplayDriver` 提供纹理显示所需的着色器功能。
- **关键成员**:
  - `shader_program_` (`uint`) -- 链接后的 OpenGL 着色器程序 ID
  - `image_texture_location_` (`int`) -- `image_texture` uniform 变量的位置，用于绑定渲染结果纹理采样器
  - `fullscreen_location_` (`int`) -- `fullscreen` uniform 变量的位置，传入视口尺寸用于坐标归一化
  - `position_attribute_location_` (`int`) -- `pos` 顶点属性的位置（缓存值）
  - `tex_coord_attribute_location_` (`int`) -- `texCoord` 顶点属性的位置（缓存值）
  - `shader_compile_attempted_` (`bool`) -- 标记着色器是否已尝试编译，防止失败后重复编译
- **关键方法**:
  - `bind(width, height)` -- 绑定着色器程序并设置 uniform 变量：将纹理采样器绑定到纹理单元 0，传入视口宽高
  - `unbind()` -- 解绑着色器程序（当前为空实现）
  - `get_position_attrib_location()` -- 获取顶点位置属性的 location，首次调用时查询并缓存
  - `get_tex_coord_attrib_location()` -- 获取纹理坐标属性的 location，首次调用时查询并缓存
  - `create_shader_if_needed()` -- 延迟创建着色器：仅在首次需要时编译并链接，查询 uniform 位置，失败则清理并标记
  - `destroy_shader()` -- 删除着色器程序并重置 ID
  - `get_shader_program()` -- 返回当前着色器程序 ID
- **静态常量**:
  - `position_attribute_name` = `"pos"` -- 顶点位置属性名称
  - `tex_coord_attribute_name` = `"texCoord"` -- 纹理坐标属性名称

## 枚举与常量

### GLSL 着色器源码（文件作用域静态常量）

- **`VERTEX_SHADER`** -- GLSL 330 顶点着色器源码：
  - 输入：`pos`（屏幕空间位置）、`texCoord`（纹理坐标）
  - Uniform：`fullscreen`（视口尺寸）
  - 功能：通过 `normalize_coordinates()` 将屏幕坐标归一化到 NDC 空间 [-1, 1]，公式为 `2.0 * (pos / fullscreen) - 1.0`
  - 输出：`gl_Position` 和插值纹理坐标 `texCoord_interp`

- **`FRAGMENT_SHADER`** -- GLSL 330 片段着色器源码：
  - 输入：插值纹理坐标 `texCoord_interp`
  - Uniform：`image_texture`（2D 纹理采样器）
  - 功能：采样纹理并应用硬编码的 Rec.709 gamma 校正（指数 0.45，即约 1/2.2）
  - 注释中标注了未来应使用 OpenColorIO 替代硬编码 gamma

## 核心函数

### `shader_print_errors(task, log, code)` (文件作用域静态函数)

- **功能**: 格式化输出着色器编译/链接错误信息
- **参数**: `task` -- 任务描述（"compile" 或 "linking"）；`log` -- GL 错误日志；`code` -- 着色器源码
- **行为**: 逐行输出着色器源码（带行号），便于定位错误

### `compile_shader_program()` (文件作用域静态函数)

- **功能**: 编译并链接完整的着色器程序
- **返回**: 成功返回程序 ID，失败返回 0
- **流程**:
  1. 创建 GL 程序对象
  2. 依次编译顶点着色器和片段着色器，失败时打印错误并返回 0
  3. 绑定片段输出 `fragColor` 到颜色附件 0
  4. 链接程序，失败时打印两个着色器的错误信息并返回 0

## 依赖关系

- **内部头文件**:
  - `util/types.h` -- 基础类型定义（`uint` 等）
  - `util/log.h` -- 日志工具（`LOG_ERROR`）
  - `util/string.h` -- 字符串工具
- **外部库**:
  - `<epoxy/gl.h>` -- OpenGL 函数加载库（提供 `glCreateProgram`、`glCompileShader` 等全部 GL API）
  - `<sstream>` -- 字符串流（错误信息格式化）
- **被引用**:
  - `src/app/opengl/display_driver.h` -- `OpenGLDisplayDriver` 包含 `OpenGLShader` 成员
  - `src/app/opengl/display_driver.cpp` -- 调用着色器的绑定/解绑方法

## 实现细节 / 关键算法

### 延迟编译与单次尝试

`create_shader_if_needed()` 实现了延迟初始化模式：着色器仅在第一次 `bind()` 调用时编译。通过 `shader_compile_attempted_` 标志确保编译/链接失败后不会在后续帧反复尝试，避免每帧重复产生错误日志和 GPU 开销。

### Gamma 校正

片段着色器中使用 `pow(rgba, vec4(0.45, 0.45, 0.45, 1.0))` 对 RGB 通道应用简化的 Rec.709 gamma 校正（alpha 通道保持不变）。指数 0.45 近似于标准 sRGB gamma 的 1/2.2。源码注释表明未来计划使用 OpenColorIO 进行更精确的色彩管理。

### 坐标归一化

顶点着色器将以像素为单位的屏幕坐标转换为 OpenGL 的 NDC（Normalized Device Coordinates）空间。公式 `2.0 * (pos / fullscreen) - 1.0` 将 [0, fullscreen] 范围映射到 [-1, 1]，使绘制的四边形恰好覆盖指定的视口区域。

### 属性位置缓存

`get_position_attrib_location()` 和 `get_tex_coord_attrib_location()` 在首次调用时通过 `glGetAttribLocation` 查询属性位置，并将结果缓存在成员变量中，后续调用直接返回缓存值，避免重复的 GL 查询开销。

## 关联文件

- `src/app/opengl/display_driver.h` / `display_driver.cpp` -- `OpenGLDisplayDriver` 持有 `OpenGLShader` 成员并在绘制流程中调用其方法
- `src/session/display_driver.h` -- 定义了显示驱动基类接口
- `src/app/cycles_standalone.cpp` -- 间接通过 `OpenGLDisplayDriver` 使用本着色器
