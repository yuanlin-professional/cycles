# sky.h - 天空纹理 SVM 节点

## 概述
`sky.h` 实现了 Cycles 的天空纹理节点(`NODE_TEX_SKY`)，提供三种物理天空模型：Preetham (1999)、Hosek-Wilkie (2012) 和 Nishita（改进型大气散射）。这些模型基于大气光学理论，可生成真实的昼间天空渲染，包括太阳光盘、天空颜色渐变、临边变暗效果等。

该节点通常用于环境照明和背景渲染，是 Cycles 中唯一依赖图像查找表(LUT)的程序化纹理（Nishita 模型使用预计算的天空 LUT）。

## 核心函数

### 通用辅助函数
- **`sky_angle_between(thetav, phiv, theta, phi)`** - 计算球面坐标系中两个方向之间的夹角

### Preetham 天空模型
- **`sky_perez_function(lam, theta, gamma)`** - Perez 全天模型函数，5 参数曲线拟合天空亮度分布
- **`sky_radiance_preetham(kg, dir, sunphi, suntheta, radiance_xyz, config_xyz)`** - 计算 Preetham 模型的天空辐射度
  - 将方向转换为球面坐标
  - 计算与太阳方向的夹角 gamma
  - 在 xyY 色彩空间中分别求值三个通道
  - 转换为 RGB 输出

### Hosek-Wilkie 天空模型
- **`sky_radiance_internal(configuration, theta, gamma)`** - Hosek-Wilkie 9 参数全天模型内部函数
  - 包含 Mie 散射项 `(1 + cos^2(gamma)) / (1 + c8^2 - 2*c8*cos(gamma))^1.5`
  - 包含天顶项 `sqrt(cos(theta))`
- **`sky_radiance_hosek(kg, dir, sunphi, suntheta, radiance_xyz, config_xyz)`** - 计算 Hosek-Wilkie 天空辐射度
  - 在 XYZ 色彩空间直接计算
  - 输出乘以 `2*pi / 683` 进行强度校正

### Nishita 改进天空模型
- **`geographical_to_direction(lat, lon)`** - 地理坐标转方向向量
- **`sky_radiance_nishita(kg, dir, path_flag, pixel_bottom, pixel_top, sky_data, texture_id)`** - 计算 Nishita 天空辐射度
  - **太阳光盘**: 当光线在太阳角直径范围内时渲染太阳，使用临边变暗系数 0.6
  - **天空**: 从预计算的 LUT 纹理采样，使用非线性 UV 映射
  - 支持太阳引导(sun guiding)的背景烘焙特殊路径

### SVM 节点入口
- **`svm_node_tex_sky(kg, path_flag, stack, node, offset)`** - SVM 节点主函数
  - 根据 `NodeSkyType` 分发到对应天空模型
  - Preetham 和 Hosek 模型从 SVM 节点数据中读取 9 个配置参数（每通道）
  - Nishita 模型读取太阳高度、旋转、角直径、强度和地平线角度

## 依赖关系
- **内部头文件**:
  - `kernel/image.h` - 图像纹理采样（`kernel_tex_image_interp`，Nishita LUT）
  - `kernel/svm/types.h` - `NodeSkyType` 枚举
  - `kernel/svm/util.h` - 栈操作和节点读取
  - `kernel/util/colorspace.h` - 色彩空间转换
  - `util/color.h` - 颜色工具函数（`xyY_to_xyz`, `xyz_to_rgb_clamped`）
- **被引用**:
  - `kernel/svm/svm.h` - 主解释器通过 `NODE_TEX_SKY` case 调用

## 实现细节

### Preetham 模型 (1999)
基于 Perez 的全天亮度模型，使用 5 个参数 `lam[0..4]` 拟合：
```
F(theta, gamma) = (1 + A * exp(B/cos(theta))) * (1 + C * exp(D*gamma) + E * cos^2(gamma))
```
在 CIE xyY 色彩空间中独立计算三个通道后转换为 RGB。适用于简单的晴天场景。

### Hosek-Wilkie 模型 (2012)
使用 9 个参数的扩展模型，增加了 Mie 散射项和天顶亮度修正：
```
F = (1 + c0*exp(c1/(cos(theta)+0.01))) * (c2 + c3*exp(c4*gamma) + c5*cos^2(gamma) + c6*mie + c7*sqrt(cos(theta)))
```
在 XYZ 色彩空间直接计算，精度高于 Preetham。

### Nishita 天空模型
使用预计算的天空查找表(LUT)，在 UV 空间中采样：
- **U 坐标**: 方位角（考虑太阳旋转偏移）
- **V 坐标**: 使用非线性映射 `sqrt(|elevation| * 2/pi) * sign(elevation) * 0.5 + 0.5`，在地平线附近提供更高的采样精度

### 太阳光盘渲染
当光线方向落入太阳角直径的一半范围内时：
1. 使用 `pixel_bottom` 和 `pixel_top` 在太阳光盘垂直方向上插值
2. 应用临边变暗(limb darkening)效果：`1 - 0.6 * (1 - sqrt(1 - (angle/half_angular)^2))`
3. 乘以太阳强度 `sun_intensity`
4. 当路径为重要性烘焙且启用太阳引导时，跳过太阳光盘渲染

### 数据读取量
天空纹理是 SVM 中数据读取量最大的节点之一：
- Preetham/Hosek 模型读取 8 个 `float4`（32 个浮点值），存储 3 组 9 参数配置
- Nishita 模型读取 3 个 `float4`（12 个浮点值）加一个纹理 ID

## 关联文件
- `kernel/svm/types.h` - `NodeSkyType` 枚举（Preetham/Hosek/单次散射/多次散射）
- `kernel/image.h` - 天空 LUT 纹理采样
- `kernel/util/colorspace.h` - CIE XYZ 到 RGB 色彩转换
- `kernel/svm/svm.h` - SVM 主解释器
