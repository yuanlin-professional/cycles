# doc - 文档与许可

## 概述

本目录存放 Cycles 渲染引擎的许可证文件和预计算工具脚本。`license/` 子目录包含 Cycles 源代码所使用的各类开源许可证全文及 SPDX 标识索引；`precompute/` 子目录包含用于生成渲染器内部查找表（LUT）的预计算 Python 脚本。这些内容在构建时由 CMake 安装到发布目录，确保最终产物附带完整的许可证信息。

## 目录结构

| 文件/目录 | 类型 | 说明 |
|-----------|------|------|
| `CMakeLists.txt` | 构建配置 | 顶层 CMake 文件，将 `license/` 子目录纳入构建 |
| `license/` | 子目录 | 许可证文件集合 |
| `license/Apache2-license.txt` | 许可证 | Apache License 2.0 全文，Cycles 源代码的默认许可证 |
| `license/BSD-3-Clause-license.txt` | 许可证 | BSD 3-Clause License 全文 |
| `license/MIT-license.txt` | 许可证 | MIT License 全文 |
| `license/Zlib-license.txt` | 许可证 | Zlib License 全文 |
| `license/SPDX-license-identifiers.txt` | 索引 | SPDX 许可证标识符列表，映射源代码中的标识符到对应的许可证文件 |
| `license/readme.txt` | 说明 | 许可证目录的说明文件，阐述默认许可证策略 |
| `license/CMakeLists.txt` | 构建配置 | 定义许可证文件列表并通过 `delayed_install` 安装到 `${CYCLES_INSTALL_PATH}/license` |
| `precompute/` | 子目录 | 预计算脚本目录 |
| `precompute/thin_film_table.py` | Python 脚本 | 薄膜干涉色彩匹配函数（CMF）查找表生成器 |

## 核心内容

### 许可证体系

Cycles 源代码默认采用 **Apache License 2.0**。此外，部分代码改编自其他来源，使用了不同但兼容的许可证。所有源文件通过 SPDX 许可证标识符（如 `SPDX-License-Identifier: Apache-2.0`）标注其适用的许可证类型。

本目录支持的四种许可证：

| SPDX 标识符 | 许可证名称 | 文件 |
|-------------|-----------|------|
| `Apache-2.0` | Apache License 2.0 | `Apache2-license.txt` |
| `BSD-3-Clause` | BSD 3-Clause License | `BSD-3-Clause-license.txt` |
| `MIT` | MIT License | `MIT-license.txt` |
| `Zlib` | Zlib License | `Zlib-license.txt` |

### 许可证安装机制

`license/CMakeLists.txt` 通过 `delayed_install()` 命令将所有许可证文件和说明文件安装到构建输出的 `${CYCLES_INSTALL_PATH}/license` 目录，确保 Cycles 的发布包中包含完整的许可证信息。

## 预计算工具

### `thin_film_table.py` - 薄膜干涉 CMF 查找表生成器

此 Python 脚本用于生成薄膜干涉（Thin Film Interference）效果所需的色彩匹配函数（Color Matching Function）查找表。薄膜干涉是光线在薄膜（如肥皂泡、油膜）表面发生干涉时产生彩虹色的物理现象。

**工作原理：**

1. **输入数据**：内置 CIE 1931 2-degree XYZ 标准观察者色彩匹配函数数据（波长范围 359-830nm）
2. **插值**：使用 `scipy.interpolate.CubicSpline` 对 CIE XYZ 数据进行三次样条插值
3. **频域重采样**：将波长域数据转换到频率域，在均匀频率网格上重采样（覆盖 0-60um，共 1022 个数据点）
4. **傅里叶变换**：对重采样后的 X、Y、Z 通道分别执行实数 FFT（`np.fft.rfft`），得到 512 个复数值
5. **输出**：生成 C 语言静态数组 `table_thin_film_cmf[512][6]`，每行包含 X/Y/Z 三通道的实部和虚部共 6 个浮点值

**运行依赖：**
- Python 3
- NumPy
- SciPy

**使用方法：**
```bash
python src/doc/precompute/thin_film_table.py
```

脚本的输出结果被写入 `src/scene/shader.tables` 文件，在编译时作为静态数据嵌入到 Cycles 渲染器的着色器模块中。该查找表在运行时被 `src/kernel/closure/bsdf_microfacet.h` 等内核闭包代码使用，用于计算薄膜干涉产生的颜色偏移。

## 依赖关系

### 上游依赖（本模块依赖）

- **Python / NumPy / SciPy**：`thin_film_table.py` 预计算脚本的运行环境
- **CIE 1931 XYZ 色彩匹配函数数据**：内嵌于脚本中的标准光谱数据

### 下游依赖（依赖本模块）

- **`src/scene/shader.tables`**：包含 `thin_film_table.py` 生成的 `table_thin_film_cmf` 静态查找表
- **`src/scene/shader.cpp`**：引用并使用薄膜干涉查找表数据
- **`src/kernel/closure/bsdf_microfacet.h`**：内核中的微面元 BSDF 闭包，使用薄膜干涉表计算反射颜色
- **`src/kernel/closure/bsdf_util.h`**：BSDF 工具函数，涉及薄膜干涉计算
- **`src/kernel/svm/closure.h`**：SVM 着色器虚拟机中的闭包节点，处理薄膜干涉参数
- **`src/kernel/osl/closures_setup.h`**：OSL 着色器闭包初始化，支持薄膜干涉参数
- **`src/kernel/types.h`**：内核类型定义，包含薄膜干涉相关的数据结构字段
- **构建系统（CMake）**：依赖 `license/CMakeLists.txt` 完成许可证文件的安装

## 参见

- `src/scene/shader.cpp` - 引用薄膜查找表的场景层着色器代码
- `src/scene/shader.tables` - 预计算生成的静态查找表数据文件
- `src/kernel/closure/bsdf_microfacet.h` - 使用薄膜干涉的微面元 BSDF 内核代码
- [SPDX License List](https://spdx.org/licenses/) - SPDX 许可证标识符官方列表
- [CIE 1931 Color Space](https://en.wikipedia.org/wiki/CIE_1931_color_space) - CIE 1931 色彩空间参考资料
