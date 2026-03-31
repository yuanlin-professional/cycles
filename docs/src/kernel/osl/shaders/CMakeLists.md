# CMakeLists.txt - 开放着色语言着色器构建配置

## 概述

该 CMake 构建脚本负责编译和安装 Cycles 渲染器的所有开放着色语言（OSL）着色器文件。它将 `.osl` 源文件编译为 `.oso` 字节码文件，并管理着色器头文件的安装。

## 宏定义/函数/类型

### CMake 变量

| 变量名 | 说明 |
|--------|------|
| `SRC_OSL` | OSL 着色器源文件列表（.osl 文件），包含所有着色器节点实现 |
| `SRC_OSL_HEADERS` | OSL 头文件列表（.h 文件），包含自定义头文件和 OSL 发行版头文件 |
| `SRC_OSL_HEADER_DIST` | 通过 `file(GLOB)` 自动发现的 OSL 发行版头文件 |
| `SRC_OSO` | 编译后的 OSO 字节码文件列表（构建时动态生成） |

### 着色器源文件清单

构建系统管理以下类别的着色器文件（共计约 80+ 个 .osl 文件）：

- **BSDF 着色器**：`node_diffuse_bsdf`、`node_glossy_bsdf`、`node_glass_bsdf`、`node_metallic_bsdf`、`node_principled_bsdf` 等
- **纹理着色器**：`node_brick_texture`、`node_checker_texture`、`node_noise_texture`、`node_voronoi_texture` 等
- **颜色工具**：`node_mix`、`node_mix_color`、`node_rgb_curves`、`node_hsv`、`node_brightness` 等
- **转换节点**：`node_convert_from_*`、`node_combine_*`、`node_separate_*` 系列
- **矢量工具**：`node_mapping`、`node_vector_math`、`node_vector_rotate`、`node_bump` 等
- **输入节点**：`node_geometry`、`node_object_info`、`node_camera`、`node_attribute` 等
- **输出节点**：`node_output_surface`、`node_output_volume`、`node_output_displacement`
- **体积着色器**：`node_scatter_volume`、`node_absorption_volume`、`node_volume_coefficients`、`node_principled_volume`
- **毛发着色器**：`node_hair_bsdf`、`node_principled_hair_bsdf`

### 头文件清单

| 头文件 | 说明 |
|--------|------|
| `int_vector_types.h` | 整数向量类型定义 |
| `node_color.h` | 颜色空间转换辅助函数 |
| `node_color_blend.h` | 颜色混合模式函数 |
| `node_fractal_voronoi.h` | 分形 Voronoi 噪声辅助函数 |
| `node_fresnel.h` | Fresnel（菲涅尔）计算辅助函数 |
| `node_hash.h` | 哈希函数 |
| `node_math.h` | 数学辅助函数 |
| `node_noise.h` | 噪声函数 |
| `node_ramp_util.h` | 渐变查找辅助函数 |
| `node_radial_tiling_shared.h` | 径向平铺共享代码 |
| `node_voronoi.h` | Voronoi 噪声辅助函数 |
| `stdcycles.h` | Cycles 标准 OSL 头文件 |

### 构建流程

1. **源文件遍历**：使用 `foreach` 循环遍历 `SRC_OSL` 中的每个 `.osl` 文件。
2. **编译规则**：为每个 `.osl` 文件创建自定义命令（`add_custom_command`），使用 OSL 编译器（`${OSL_COMPILER}`）将其编译为 `.oso` 文件：
   - 编译选项：`-q`（安静模式）、`-O2`（二级优化）
   - 包含路径：当前源目录和 OSL 着色器安装目录
   - 输出路径：对应的构建目录
3. **构建目标**：创建 `cycles_osl_shaders` 自定义目标，依赖所有 `.oso` 文件、头文件和 OSL 编译器。
4. **安装规则**：
   - `.oso` 字节码文件安装到 `${CYCLES_INSTALL_PATH}/shader`
   - 头文件也安装到同一目录（供运行时 OSL 编译使用）

## 依赖关系

- **外部依赖**：
  - `OSL_COMPILER` - 开放着色语言编译器可执行文件
  - `OSL_SHADER_DIR` - OSL 发行版的着色器头文件目录
- **内部依赖**：所有 `.osl` 文件依赖于 `SRC_OSL_HEADERS` 中列出的全部头文件
- **构建产物**：`.oso` 编译后字节码文件，安装到 Cycles 着色器目录
- **解决方案文件夹**：通过 `cycles_set_solution_folder` 设置 IDE 项目组织
