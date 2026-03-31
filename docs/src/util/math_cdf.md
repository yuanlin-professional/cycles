# math_cdf.h / math_cdf.cpp - 累积分布函数计算与反转

## 概述

`math_cdf.h` 和 `math_cdf.cpp` 提供累积分布函数 (CDF) 的计算和反转工具。CDF 是重要性采样的基础数学工具，在 Cycles 中用于像素滤波器的采样分布构建（如 Gaussian、Blackman-Harris 等滤波器）以及相机景深采样。

## 核心函数

### math_cdf.h（模板函数，头文件内联）

| 函数 | 说明 |
|------|------|
| `util_cdf_evaluate(resolution, from, to, functor, &cdf)` | 对给定函子在 [from, to] 范围内以指定分辨率采样，计算其归一化 CDF |
| `util_cdf_inverted(resolution, from, to, functor, symmetric, &inv_cdf)` | 先计算 CDF 再反转，组合函数 |

### math_cdf.cpp（非模板函数）

| 函数 | 说明 |
|------|------|
| `util_cdf_invert(resolution, from, to, &cdf, symmetric, &inv_cdf)` | 将预计算的 CDF 反转为逆 CDF 查找表，支持对称模式 |

## 依赖关系

- **内部头文件**: `util/math_base.h`, `util/vector.h`（CDF 存储在 `vector<float>` 中）
- **cpp 额外依赖**: `util/algorithm.h`（使用 `upper_bound` 进行二分查找）
- **被引用**: `scene/film.cpp`（像素滤波器采样表）, `scene/camera.cpp`（景深光圈采样）

## 实现细节 / 关键算法

1. **CDF 计算 (`util_cdf_evaluate`)**:
   - 在 [from, to] 范围内均匀采样 `resolution` 个点
   - 对函子的绝对值进行累积求和
   - 最后归一化使 CDF 范围为 [0, 1]
   - CDF 数组大小为 `resolution + 1`，首元素为 0，末元素强制设为 1

2. **CDF 反转 (`util_cdf_invert`)**:
   - **非对称模式**: 对每个均匀分布的输出值 x，使用 `upper_bound` 二分查找在 CDF 表中的位置，再线性插值得到对应的输入值
   - **对称模式**: 仅计算半侧 CDF，然后镜像对称填充。用于像素滤波器等关于中心对称的分布。产生形如 `0.5 * (1 +/- y)` 的对称查找表

3. **分辨率的微妙处理**: `util_cdf_inverted` 中，CDF 计算使用 `resolution - 1` 而非 `resolution`，这是为了兼容像素滤波器的旧代码，确保回归测试不会失败。

## 关联文件

- `util/vector.h` — `vector<float>` 容器（标准 std::vector 的 Cycles 别名）
- `util/algorithm.h` — `upper_bound` 二分查找
- `scene/film.cpp` — 像素滤波器 CDF 表的主要使用者
- `scene/camera.cpp` — 相机光圈采样表
