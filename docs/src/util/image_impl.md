# image_impl.h - 图像缩放算法模板实现

## 概述
本文件是 `image.h` 中声明的图像缩放函数 `util_image_resize_pixels` 的模板实现文件。提供了基于 box 滤波器的降采样算法，包括像素读取、采样核计算和完整的降采样流程。该文件由 `image.h` 末尾自动包含（`#include "util/image_impl.h"`）。

## 类与结构体
无。本文件仅包含匿名命名空间中的模板辅助函数和外部模板实现。

## 核心函数

### 内部辅助函数（匿名命名空间）

| 函数 | 说明 |
|------|------|
| `const T *util_image_read<T>(...)` | 根据 (x, y) 坐标和分量数读取像素指针 |
| `void util_image_downscale_sample<T>(...)` | 对给定位置进行 box 滤波采样，核大小由缩放因子决定 |
| `void util_image_downscale_pixels<T>(...)` | 遍历输出图像所有像素并调用采样函数完成降采样 |

### 公开模板函数

| 函数 | 说明 |
|------|------|
| `void util_image_resize_pixels<T>(...)` | 主入口函数，接受输入像素数据和缩放因子，输出缩放后的像素数组 |

#### `util_image_resize_pixels` 工作流程：
1. 若缩放因子为 1.0，直接复制输入数据
2. 计算输出尺寸（最小为 1x1）
3. 若缩放因子 < 1.0，调用 `util_image_downscale_pixels` 降采样
4. 放大（缩放因子 > 1.0）标记为 TODO，暂未实现

#### box 滤波器采样细节：
- 核大小 = `round(1/scale_factor)`
- 对核覆盖范围内的所有像素先转为 float 累加
- 取平均值后再转回目标类型
- 自动处理图像边界（越界像素不参与计算）

## 依赖关系
- **内部头文件**: `util/image.h`（父文件）, `util/math_base.h`
- **被引用**: 通过 `util/image.h` 间接包含

## 关联文件
- `util/image.h` - 声明和类型转换工具
- `util/half.h` - `half` 类型支持
