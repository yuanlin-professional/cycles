# image.h - CPU 设备纹理图像插值实现

## 概述

本文件实现了 Cycles 渲染器 CPU 后端的 2D 纹理图像采样与插值逻辑。通过模板类 `TextureInterpolator` 支持多种像素数据类型（float、half、uchar、uint16_t 及其 4 通道变体），并提供最近邻、双线性和三次样条三种插值模式，以及重复、裁剪、延伸和镜像四种边界扩展模式。所有模板函数放在匿名命名空间中，防止不同指令集内核间的符号冲突。

## 核心函数/宏定义

### `TextureInterpolator<TexT, OutT>` 模板类

#### 像素读取函数 `read()`
重载多种像素类型的读取：
- `float4` / `float` — 直接返回
- `uchar4` / `uchar` — 除以 255.0 归一化
- `half4` / `half` — 通过 `half4_to_float4_image` / `half_to_float_image` 转换
- `uint16_t` / `ushort4` — 除以 65535.0 归一化
- `read(data, x, y, width, height)` — 2D 纹理寻址读取
- `read_clip(data, x, y, width, height)` — 带边界检查的读取，越界返回透明黑

#### 边界环绕函数
- **`wrap_periodic(x, width)`**: 周期重复环绕（取模），处理负值
- **`wrap_clamp(x, width)`**: 钳位到 [0, width-1]
- **`wrap_mirror(x, width)`**: 镜像反射环绕

#### 2D 插值函数
- **`interp_closest(info, x, y)`**: 最近邻插值，直接取最近像素
- **`interp_linear(info, x, y)`**: 双线性插值，使用 -0.5 偏移使采样点居中，对四个相邻像素进行加权混合
- **`interp_cubic(info, x, y)`**: 三次样条插值，使用 4x4 像素邻域和三次样条权重
- **`interp(info, x, y)`**: 根据 `TextureInfo::interpolation` 分派到对应插值函数

### `SET_CUBIC_SPLINE_WEIGHTS(u, t)` 宏
计算三次样条插值的四个权重系数，基于参数 t (0~1)。

### `frac(x, ix)` 函数
将浮点数分解为整数部分（存入 `*ix`）和小数部分（返回值），正确处理负数。

### `kernel_tex_image_interp(kg, id, x, y)`
顶层纹理采样入口函数。根据 `TextureInfo::data_type` 选择合适的模板实例化：
- 单通道类型（HALF、BYTE、USHORT、FLOAT）返回灰度值扩展为 `float4`
- 四通道类型（HALF4、BYTE4、USHORT4、FLOAT4）直接返回 `float4`

## 依赖关系

- **内部头文件**:
  - `kernel/device/cpu/compat.h` (CPU 兼容层)
  - `kernel/device/cpu/globals.h` (全局数据，提供 `kernel_data_fetch`)
  - `util/half.h` (半精度浮点转换)
- **被引用**:
  - `src/kernel/image.h` (通用图像采样接口)
  - `src/kernel/osl/services.cpp` (OSL 纹理服务)
  - `src/kernel/device/cpu/kernel_arch_impl.h` (CPU 内核实现模板)

## 实现细节 / 关键算法

### 匿名命名空间隔离
所有模板函数放在匿名命名空间 `namespace { }` 中，确保不同编译单元（如 `kernel.cpp` 和 `kernel_avx2.cpp`）使用各自的内联副本，避免因不同指令集优化导致的符号冲突。

### 三次样条插值
使用 Catmull-Rom 变体的三次样条，权重公式为：
- `u[0] = (((-1/6)*t + 0.5)*t - 0.5)*t + 1/6`
- `u[1] = ((0.5*t - 1)*t)*t + 2/3`
- `u[2] = ((-0.5*t + 0.5)*t + 0.5)*t + 1/6`
- `u[3] = (1/6)*t*t*t`

在 4x4 邻域上进行二维卷积，通过宏 `DATA(x,y)` 和 `TERM(col)` 简化矩阵乘法代码。

### CLIP 扩展模式优化
对于 CLIP 模式，线性插值提前检查采样窗口是否完全在图像外部（`ix < -1 || ix >= width`），三次插值检查更大范围（`ix < -2 || ix > width`），避免不必要的像素读取。

## 关联文件

- `src/kernel/device/gpu/image.h` — GPU 设备的纹理采样实现（使用硬件纹理单元）
- `src/kernel/image.h` — 内核层面统一纹理采样接口
- `src/kernel/device/cpu/globals.h` — 提供 `kernel_data_fetch` 宏获取纹理信息
