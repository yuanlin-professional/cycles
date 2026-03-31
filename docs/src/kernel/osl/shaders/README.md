# kernel/osl/shaders - OSL 着色器节点库

## 概述

本目录包含 Cycles 渲染器的全部开放着色语言 (OSL) 着色器节点实现。这些 `.osl` 文件在 CPU 渲染模式下被编译执行，提供与着色器虚拟机 (SVM) 等价的着色功能。每个文件对应一个着色器节点。

目录中共有 98 个 `.osl` 着色器文件和 13 个 `.h` 辅助头文件。着色器使用 `#include "stdcycles.h"` 引入 Cycles 标准宏定义，部分着色器还依赖专用工具头文件（如 `node_fresnel.h`、`node_color.h`、`node_noise.h` 等）。

OSL 着色器文件中的主入口根据功能声明为不同类型：
- `shader` -- 通用着色器节点（大多数节点）
- `surface` -- 表面着色器（如 `node_output_surface`、`node_bump`、`node_set_normal`）
- `volume` -- 体积着色器（如 `node_output_volume`）
- `displacement` -- 置换着色器（如 `node_output_displacement`）

## 着色器节点总表

| 文件名 | 节点名称 | 功能说明 |
|--------|----------|----------|
| `node_absorption_volume.osl` | node_absorption_volume | 吸收体积：根据颜色和密度计算体积光吸收闭包 |
| `node_add_closure.osl` | node_add_closure | 闭包相加：将两个闭包叠加为一个闭包输出 |
| `node_ambient_occlusion.osl` | node_ambient_occlusion | 环境光遮蔽：计算局部遮蔽因子和遮蔽颜色 |
| `node_attribute.osl` | node_attribute | 属性：按名称读取几何体自定义属性数据 |
| `node_background.osl` | node_background | 背景：生成环境光/世界背景闭包 |
| `node_bevel.osl` | node_bevel | 倒角：通过光线投射近似计算倒角法线 |
| `node_blackbody.osl` | node_blackbody | 黑体辐射：根据温度计算黑体辐射颜色 |
| `node_brick_texture.osl` | node_brick_texture | 砖墙纹理：生成程序化砖块/砂浆图案 |
| `node_brightness.osl` | node_brightness | 亮度/对比度：调整颜色的亮度和对比度 |
| `node_bump.osl` | node_bump | 凹凸映射：基于高度差采样计算凹凸法线偏移 |
| `node_camera.osl` | node_camera | 相机数据：输出视图向量、Z 深度和视距 |
| `node_checker_texture.osl` | node_checker_texture | 棋盘格纹理：生成三维棋盘格图案 |
| `node_clamp.osl` | node_clamp | 钳制：将值限制在最小/最大范围内 |
| `node_combine_color.osl` | node_combine_color | 合并颜色：从 RGB/HSV/HSL 分量合成颜色 |
| `node_combine_xyz.osl` | node_combine_xyz | 合并 XYZ：从三个浮点值组合为向量 |
| `node_convert_from_color.osl` | node_convert_from_color | 颜色转换：将颜色类型转换为浮点/整数/向量/点/法线 |
| `node_convert_from_float.osl` | node_convert_from_float | 浮点转换：将浮点类型转换为整数/颜色/向量/点/法线 |
| `node_convert_from_int.osl` | node_convert_from_int | 整数转换：将整数类型转换为浮点/颜色/向量/点/法线 |
| `node_convert_from_normal.osl` | node_convert_from_normal | 法线转换：将法线类型转换为浮点/整数/向量/颜色/点 |
| `node_convert_from_point.osl` | node_convert_from_point | 点转换：将点类型转换为浮点/整数/向量/颜色/法线 |
| `node_convert_from_string.osl` | node_convert_from_string | 字符串转换：字符串类型到其他类型的占位转换 |
| `node_convert_from_vector.osl` | node_convert_from_vector | 向量转换：将向量类型转换为浮点/整数/颜色/点/法线 |
| `node_diffuse_bsdf.osl` | node_diffuse_bsdf | 漫反射 BSDF：Lambert/Oren-Nayar 漫反射闭包 |
| `node_displacement.osl` | node_displacement | 置换：根据高度图生成置换向量 |
| `node_emission.osl` | node_emission | 自发光：生成自发光/发射闭包 |
| `node_environment_texture.osl` | node_environment_texture | 环境纹理：采样环境贴图（等距柱状投影/镜面球） |
| `node_float_curve.osl` | node_float_curve | 浮点曲线：通过查找表映射浮点值 |
| `node_fresnel.osl` | node_fresnel | 菲涅尔：计算基于 IOR 的菲涅尔反射系数 |
| `node_gabor_texture.osl` | node_gabor_texture | Gabor 纹理：基于稀疏 Gabor 卷积的程序化噪波纹理 |
| `node_gamma.osl` | node_gamma | 伽马校正：对颜色进行伽马幂次变换 |
| `node_geometry.osl` | node_geometry | 几何数据：输出位置、法线、切线、参数坐标等几何信息 |
| `node_glass_bsdf.osl` | node_glass_bsdf | 玻璃 BSDF：带薄膜干涉的玻璃微面反射/折射闭包 |
| `node_glossy_bsdf.osl` | node_glossy_bsdf | 光泽 BSDF：各向异性微面高光反射闭包 |
| `node_gradient_texture.osl` | node_gradient_texture | 渐变纹理：生成线性/二次/对角/球形/径向渐变图案 |
| `node_hair_bsdf.osl` | node_hair_bsdf | 毛发 BSDF：毛发反射/透射闭包 |
| `node_hair_info.osl` | node_hair_info | 毛发信息：输出毛发截距、长度、粗细等属性 |
| `node_holdout.osl` | node_holdout | 遮罩保留：生成遮罩（holdout）闭包 |
| `node_hsv.osl` | node_hsv | 色相/饱和度/明度：调整颜色的 HSV 分量 |
| `node_ies_light.osl` | node_ies_light | IES 灯光：根据 IES 配光文件计算灯光强度分布 |
| `node_image_texture.osl` | node_image_texture | 图像纹理：采样外部图像纹理文件 |
| `node_invert.osl` | node_invert | 反转：对颜色进行反转 (1-Color) |
| `node_layer_weight.osl` | node_layer_weight | 层权重：输出菲涅尔和面向权重因子 |
| `node_light_falloff.osl` | node_light_falloff | 光衰减：计算二次/线性/恒定光衰减 |
| `node_light_path.osl` | node_light_path | 光线路径：输出当前光线类型和深度信息 |
| `node_magic_texture.osl` | node_magic_texture | 魔术纹理：生成基于三角函数的万花筒图案 |
| `node_mapping.osl` | node_mapping | 映射：对向量进行平移/旋转/缩放变换 |
| `node_map_range.osl` | node_map_range | 范围映射：将浮点值从一个范围重映射到另一个范围 |
| `node_math.osl` | node_math | 数学：执行各种标量数学运算（加/减/乘/除/三角函数等） |
| `node_metallic_bsdf.osl` | node_metallic_bsdf | 金属 BSDF：基于物理的金属微面反射闭包（含导体菲涅尔和薄膜干涉） |
| `node_mix.osl` | node_mix | 混合：多种颜色混合模式（叠加/屏幕/柔光/线性光等） |
| `node_mix_closure.osl` | node_mix_closure | 闭包混合：按因子在两个闭包之间线性插值 |
| `node_mix_color.osl` | node_mix_color | 颜色混合：基于混合模式混合两个颜色 |
| `node_mix_float.osl` | node_mix_float | 浮点混合：按因子在两个浮点值之间线性插值 |
| `node_mix_vector.osl` | node_mix_vector | 向量混合：按标量因子在两个向量之间线性插值 |
| `node_mix_vector_non_uniform.osl` | node_mix_vector_non_uniform | 向量非均匀混合：按向量因子在两个向量之间逐分量插值 |
| `node_noise_texture.osl` | node_noise_texture | 噪波纹理：生成多维度分形噪波图案 |
| `node_normal.osl` | node_normal | 法线：输出法线方向并计算点积 |
| `node_normal_map.osl` | node_normal_map | 法线贴图：从法线贴图颜色转换为世界/物体/切线空间法线 |
| `node_object_info.osl` | node_object_info | 物体信息：输出物体位置、颜色、Alpha、索引和随机值 |
| `node_output_displacement.osl` | node_output_displacement | 置换输出：将置换向量应用到表面位置 |
| `node_output_surface.osl` | node_output_surface | 表面输出：将闭包赋值给最终表面着色结果 (Ci) |
| `node_output_volume.osl` | node_output_volume | 体积输出：将闭包赋值给最终体积着色结果 (Ci) |
| `node_particle_info.osl` | node_particle_info | 粒子信息：输出粒子索引、年龄、寿命、位置、速度等属性 |
| `node_point_info.osl` | node_point_info | 点信息：输出点云的位置、半径和随机值 |
| `node_principled_bsdf.osl` | node_principled_bsdf | 原理化 BSDF：Disney/GGX 多层统一材质模型（支持次表面、金属、高光、光泽、清漆、透射等） |
| `node_principled_hair_bsdf.osl` | node_principled_hair_bsdf | 原理化毛发 BSDF：基于物理的毛发散射模型 |
| `node_principled_volume.osl` | node_principled_volume | 原理化体积：统一体积材质（密度、吸收、散射、自发光和黑体辐射） |
| `node_radial_tiling.osl` | node_radial_tiling | 径向平铺：径向/旋转对称的程序化纹理平铺 |
| `node_ray_portal_bsdf.osl` | node_ray_portal_bsdf | 光线传送门 BSDF：将光线传送到指定位置和方向 |
| `node_refraction_bsdf.osl` | node_refraction_bsdf | 折射 BSDF：微面折射闭包 |
| `node_rgb_curves.osl` | node_rgb_curves | RGB 曲线：通过查找表逐通道调整 RGB 曲线 |
| `node_rgb_ramp.osl` | node_rgb_ramp | RGB 渐变：通过颜色/Alpha 查找表映射浮点输入 |
| `node_rgb_to_bw.osl` | node_rgb_to_bw | 彩色转黑白：根据感知亮度公式将 RGB 转换为灰度值 |
| `node_scatter_volume.osl` | node_scatter_volume | 散射体积：根据颜色、密度和各向异性生成体积散射闭包 |
| `node_separate_color.osl` | node_separate_color | 分离颜色：将颜色拆分为 RGB/HSV/HSL 各分量 |
| `node_separate_xyz.osl` | node_separate_xyz | 分离 XYZ：将向量拆分为 X、Y、Z 分量 |
| `node_set_normal.osl` | node_set_normal | 设置法线：覆盖着色法线方向 |
| `node_sheen_bsdf.osl` | node_sheen_bsdf | 光泽 BSDF（织物）：微纤维/Charlie 分布的织物光泽闭包 |
| `node_sky_texture.osl` | node_sky_texture | 天空纹理：物理天空模型（Preetham/Hosek-Wilkie/Nishita） |
| `node_subsurface_scattering.osl` | node_subsurface_scattering | 次表面散射：BSSRDF 随机游走次表面闭包 |
| `node_tangent.osl` | node_tangent | 切线：生成 UV 贴图或径向方向的切线 |
| `node_texture_coordinate.osl` | node_texture_coordinate | 纹理坐标：输出生成坐标/UV/物体/相机/窗口/反射等坐标 |
| `node_toon_bsdf.osl` | node_toon_bsdf | 卡通 BSDF：卡通风格漫反射/光泽闭包 |
| `node_translucent_bsdf.osl` | node_translucent_bsdf | 半透明 BSDF：背光透射漫反射闭包 |
| `node_transparent_bsdf.osl` | node_transparent_bsdf | 透明 BSDF：完全透明闭包 |
| `node_uv_map.osl` | node_uv_map | UV 贴图：按层名称读取 UV 坐标 |
| `node_value.osl` | node_value | 常量值：输出常量浮点/向量/颜色值 |
| `node_vector_curves.osl` | node_vector_curves | 向量曲线：通过查找表逐分量调整向量曲线 |
| `node_vector_displacement.osl` | node_vector_displacement | 向量置换：根据向量贴图在切线/物体/世界空间进行置换 |
| `node_vector_map_range.osl` | node_vector_map_range | 向量范围映射：将向量各分量从一个范围重映射到另一个范围 |
| `node_vector_math.osl` | node_vector_math | 向量数学：执行各种向量数学运算（加/减/叉积/点积/归一化等） |
| `node_vector_rotate.osl` | node_vector_rotate | 向量旋转：绕轴或按欧拉角旋转向量 |
| `node_vector_transform.osl` | node_vector_transform | 向量变换：在世界/物体/相机空间之间变换向量/点/法线 |
| `node_vertex_color.osl` | node_vertex_color | 顶点颜色：读取网格顶点颜色属性 |
| `node_volume_coefficients.osl` | node_volume_coefficients | 体积系数：直接指定吸收/散射/发射系数的体积闭包 |
| `node_voronoi_texture.osl` | node_voronoi_texture | Voronoi 纹理：生成多维度 Voronoi/Worley 噪波图案 |
| `node_wave_texture.osl` | node_wave_texture | 波纹纹理：生成带/环形波纹图案 |
| `node_wavelength.osl` | node_wavelength | 波长转颜色：将可见光波长转换为 RGB 颜色 |
| `node_white_noise_texture.osl` | node_white_noise_texture | 白噪声纹理：生成多维度纯随机白噪声 |
| `node_wireframe.osl` | node_wireframe | 线框：计算三角面边缘的线框遮罩因子 |

## 辅助头文件

| 文件名 | 说明 |
|--------|------|
| `stdcycles.h` | Cycles 标准宏定义和常量（凹凸滤波宽度等） |
| `node_color.h` | 颜色空间转换工具函数（RGB/HSV/HSL 互转） |
| `node_color_blend.h` | 颜色混合模式实现（叠加、屏幕、柔光等） |
| `node_fresnel.h` | 菲涅尔计算工具函数 |
| `node_math.h` | 数学工具宏和安全除法等辅助函数 |
| `node_noise.h` | Perlin 噪波和分形噪波核心算法 |
| `node_hash.h` | 哈希函数实现（用于白噪声等） |
| `node_ramp_util.h` | RGB/浮点曲线查找表插值工具 |
| `node_fractal_voronoi.h` | 分形 Voronoi 噪波算法 |
| `node_voronoi.h` | Voronoi 细胞噪波核心算法 |
| `node_scatter.h` | 体积散射相位函数辅助（Henyey-Greenstein 等） |
| `node_radial_tiling_shared.h` | 径向平铺的圆角多边形计算函数 |
| `int_vector_types.h` | 整数向量类型定义 |

## 节点分类

### 纹理节点

生成程序化纹理或采样外部纹理图像的节点。

| 文件名 | 节点名称 | 功能说明 |
|--------|----------|----------|
| `node_brick_texture.osl` | node_brick_texture | 砖墙纹理 |
| `node_checker_texture.osl` | node_checker_texture | 棋盘格纹理 |
| `node_environment_texture.osl` | node_environment_texture | 环境纹理（HDR 全景） |
| `node_gabor_texture.osl` | node_gabor_texture | Gabor 噪波纹理 |
| `node_gradient_texture.osl` | node_gradient_texture | 渐变纹理 |
| `node_ies_light.osl` | node_ies_light | IES 灯光配光纹理 |
| `node_image_texture.osl` | node_image_texture | 图像纹理 |
| `node_magic_texture.osl` | node_magic_texture | 魔术纹理 |
| `node_noise_texture.osl` | node_noise_texture | 噪波纹理 |
| `node_radial_tiling.osl` | node_radial_tiling | 径向平铺纹理 |
| `node_sky_texture.osl` | node_sky_texture | 天空纹理 |
| `node_voronoi_texture.osl` | node_voronoi_texture | Voronoi 纹理 |
| `node_wave_texture.osl` | node_wave_texture | 波纹纹理 |
| `node_white_noise_texture.osl` | node_white_noise_texture | 白噪声纹理 |

### 颜色节点

处理颜色的调整、转换和混合。

| 文件名 | 节点名称 | 功能说明 |
|--------|----------|----------|
| `node_blackbody.osl` | node_blackbody | 黑体辐射颜色 |
| `node_brightness.osl` | node_brightness | 亮度/对比度调整 |
| `node_combine_color.osl` | node_combine_color | 合并颜色分量 |
| `node_gamma.osl` | node_gamma | 伽马校正 |
| `node_hsv.osl` | node_hsv | 色相/饱和度/明度调整 |
| `node_invert.osl` | node_invert | 颜色反转 |
| `node_mix.osl` | node_mix | 颜色混合（多种混合模式） |
| `node_mix_color.osl` | node_mix_color | 颜色混合节点 |
| `node_rgb_curves.osl` | node_rgb_curves | RGB 曲线调整 |
| `node_rgb_ramp.osl` | node_rgb_ramp | RGB 渐变映射 |
| `node_rgb_to_bw.osl` | node_rgb_to_bw | 彩色转灰度 |
| `node_separate_color.osl` | node_separate_color | 分离颜色分量 |
| `node_wavelength.osl` | node_wavelength | 波长转颜色 |

### 向量节点

向量的组合、拆分、变换和数学运算。

| 文件名 | 节点名称 | 功能说明 |
|--------|----------|----------|
| `node_bump.osl` | node_bump | 凹凸映射（法线偏移） |
| `node_combine_xyz.osl` | node_combine_xyz | 合并 XYZ 分量 |
| `node_displacement.osl` | node_displacement | 置换向量生成 |
| `node_mapping.osl` | node_mapping | 向量映射变换 |
| `node_normal.osl` | node_normal | 法线方向和点积 |
| `node_normal_map.osl` | node_normal_map | 法线贴图解码 |
| `node_separate_xyz.osl` | node_separate_xyz | 分离 XYZ 分量 |
| `node_set_normal.osl` | node_set_normal | 设置着色法线 |
| `node_vector_curves.osl` | node_vector_curves | 向量曲线调整 |
| `node_vector_displacement.osl` | node_vector_displacement | 向量置换 |
| `node_vector_map_range.osl` | node_vector_map_range | 向量范围映射 |
| `node_vector_math.osl` | node_vector_math | 向量数学运算 |
| `node_vector_rotate.osl` | node_vector_rotate | 向量旋转 |
| `node_vector_transform.osl` | node_vector_transform | 向量空间变换 |

### 数学节点

标量数学运算和值处理。

| 文件名 | 节点名称 | 功能说明 |
|--------|----------|----------|
| `node_clamp.osl` | node_clamp | 值钳制 |
| `node_float_curve.osl` | node_float_curve | 浮点曲线映射 |
| `node_map_range.osl` | node_map_range | 范围重映射 |
| `node_math.osl` | node_math | 标量数学运算 |
| `node_mix_float.osl` | node_mix_float | 浮点线性插值 |
| `node_mix_vector.osl` | node_mix_vector | 向量线性插值 |
| `node_mix_vector_non_uniform.osl` | node_mix_vector_non_uniform | 向量非均匀插值 |

### 闭包/BSDF 节点

双向散射分布函数 (BSDF)、体积散射及其他着色闭包。

| 文件名 | 节点名称 | 功能说明 |
|--------|----------|----------|
| `node_absorption_volume.osl` | node_absorption_volume | 吸收体积闭包 |
| `node_add_closure.osl` | node_add_closure | 闭包相加 |
| `node_background.osl` | node_background | 背景环境光闭包 |
| `node_diffuse_bsdf.osl` | node_diffuse_bsdf | 漫反射 BSDF |
| `node_emission.osl` | node_emission | 自发光闭包 |
| `node_glass_bsdf.osl` | node_glass_bsdf | 玻璃 BSDF |
| `node_glossy_bsdf.osl` | node_glossy_bsdf | 光泽（高光）BSDF |
| `node_hair_bsdf.osl` | node_hair_bsdf | 毛发 BSDF |
| `node_holdout.osl` | node_holdout | 遮罩保留闭包 |
| `node_metallic_bsdf.osl` | node_metallic_bsdf | 金属 BSDF |
| `node_mix_closure.osl` | node_mix_closure | 闭包混合 |
| `node_principled_bsdf.osl` | node_principled_bsdf | 原理化 BSDF（统一材质模型） |
| `node_principled_hair_bsdf.osl` | node_principled_hair_bsdf | 原理化毛发 BSDF |
| `node_principled_volume.osl` | node_principled_volume | 原理化体积 |
| `node_ray_portal_bsdf.osl` | node_ray_portal_bsdf | 光线传送门 BSDF |
| `node_refraction_bsdf.osl` | node_refraction_bsdf | 折射 BSDF |
| `node_scatter_volume.osl` | node_scatter_volume | 散射体积闭包 |
| `node_sheen_bsdf.osl` | node_sheen_bsdf | 光泽（织物）BSDF |
| `node_subsurface_scattering.osl` | node_subsurface_scattering | 次表面散射 BSSRDF |
| `node_toon_bsdf.osl` | node_toon_bsdf | 卡通 BSDF |
| `node_translucent_bsdf.osl` | node_translucent_bsdf | 半透明 BSDF |
| `node_transparent_bsdf.osl` | node_transparent_bsdf | 透明 BSDF |
| `node_volume_coefficients.osl` | node_volume_coefficients | 体积系数（直接指定吸收/散射/发射） |

### 输入节点

从场景、物体和几何体获取信息的节点。

| 文件名 | 节点名称 | 功能说明 |
|--------|----------|----------|
| `node_ambient_occlusion.osl` | node_ambient_occlusion | 环境光遮蔽 |
| `node_attribute.osl` | node_attribute | 自定义属性读取 |
| `node_bevel.osl` | node_bevel | 倒角法线 |
| `node_camera.osl` | node_camera | 相机数据 |
| `node_fresnel.osl` | node_fresnel | 菲涅尔系数 |
| `node_geometry.osl` | node_geometry | 几何数据 |
| `node_hair_info.osl` | node_hair_info | 毛发信息 |
| `node_layer_weight.osl` | node_layer_weight | 层权重 |
| `node_light_falloff.osl` | node_light_falloff | 光衰减 |
| `node_light_path.osl` | node_light_path | 光线路径 |
| `node_object_info.osl` | node_object_info | 物体信息 |
| `node_particle_info.osl` | node_particle_info | 粒子信息 |
| `node_point_info.osl` | node_point_info | 点云信息 |
| `node_tangent.osl` | node_tangent | 切线方向 |
| `node_texture_coordinate.osl` | node_texture_coordinate | 纹理坐标 |
| `node_uv_map.osl` | node_uv_map | UV 贴图坐标 |
| `node_value.osl` | node_value | 常量值 |
| `node_vertex_color.osl` | node_vertex_color | 顶点颜色 |
| `node_wireframe.osl` | node_wireframe | 线框遮罩 |

### 输出节点

着色器树的最终输出节点。

| 文件名 | 节点名称 | 功能说明 |
|--------|----------|----------|
| `node_output_displacement.osl` | node_output_displacement | 置换输出（应用 P 偏移） |
| `node_output_surface.osl` | node_output_surface | 表面输出（赋值给 Ci） |
| `node_output_volume.osl` | node_output_volume | 体积输出（赋值给 Ci） |

### 转换节点

在不同数据类型之间进行转换。

| 文件名 | 节点名称 | 功能说明 |
|--------|----------|----------|
| `node_convert_from_color.osl` | node_convert_from_color | 颜色 -> 其他类型 |
| `node_convert_from_float.osl` | node_convert_from_float | 浮点 -> 其他类型 |
| `node_convert_from_int.osl` | node_convert_from_int | 整数 -> 其他类型 |
| `node_convert_from_normal.osl` | node_convert_from_normal | 法线 -> 其他类型 |
| `node_convert_from_point.osl` | node_convert_from_point | 点 -> 其他类型 |
| `node_convert_from_string.osl` | node_convert_from_string | 字符串 -> 其他类型 |
| `node_convert_from_vector.osl` | node_convert_from_vector | 向量 -> 其他类型 |

## 依赖关系

### 上游依赖（本模块依赖）

- **OSL 标准库** -- 开放着色语言运行时及其内置函数（`diffuse()`、`microfacet()`、`bssrdf()` 等闭包原语）
- **`stdcycles.h`** -- Cycles 自定义 OSL 头文件，定义 `BUMP_FILTER_WIDTH` 等宏常量
- **辅助头文件** -- 本目录下的 `.h` 文件提供噪波、颜色转换、菲涅尔等可复用算法

### 下游依赖（依赖本模块）

- **`kernel/osl/`** -- OSL 运行时集成层，负责编译和执行本目录中的 `.osl` 着色器
- **`scene/`** -- 着色器节点编译系统，将节点图编译为 OSL 着色器调用序列

## 参见

- [kernel/osl OSL 集成](../README.md)
- [kernel/svm 着色器虚拟机 (SVM)](../../svm/README.md)
