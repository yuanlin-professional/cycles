# convert.h - 数据类型转换节点的着色器虚拟机实现

## 概述

本文件实现了着色器虚拟机（SVM）中的类型转换（Convert）节点。该节点在浮点数（Float）、整数（Int）、向量（Vector/float3）和颜色（Color/float3）四种数据类型之间进行相互转换。这是着色器节点图中隐式和显式类型转换的底层实现。

## 核心函数

### `svm_node_convert`

```c
ccl_device_noinline void svm_node_convert(
    KernelGlobals kg,
    ccl_private float *stack,
    const uint type,
    const uint from,
    const uint to)
```

**功能**：在不同数据类型之间进行转换。

**参数**：
- `kg`：内核全局数据（用于颜色到灰度的转换）
- `stack`：SVM 栈指针
- `type`：转换类型（`NodeConvert` 枚举）
- `from`：源数据的栈偏移量
- `to`：目标数据的栈偏移量

**支持的转换类型**：

| 枚举值 | 方向 | 转换方法 |
|--------|------|----------|
| `NODE_CONVERT_FI` | Float -> Int | `float_to_int(f)` |
| `NODE_CONVERT_FV` | Float -> Vector | `(f, f, f)` 扩展为三分量 |
| `NODE_CONVERT_CF` | Color -> Float | `linear_rgb_to_gray` 感知亮度 |
| `NODE_CONVERT_CI` | Color -> Int | `linear_rgb_to_gray` 后取整 |
| `NODE_CONVERT_VF` | Vector -> Float | `average(f)` 三分量均值 |
| `NODE_CONVERT_VI` | Vector -> Int | `average(f)` 后取整 |
| `NODE_CONVERT_IF` | Int -> Float | 直接类型转换 |
| `NODE_CONVERT_IV` | Int -> Vector | 整数转浮点后扩展为三分量 |

## 依赖关系

- **内部头文件**：`kernel/svm/util.h`、`kernel/util/colorspace.h`
- **被引用**：`kernel/svm/svm.h`（SVM 主调度器）

## 实现细节 / 关键算法

- **颜色与向量到标量的不同策略**：
  - **颜色到浮点**（`CF`）使用 `linear_rgb_to_gray`，这是基于人眼感知的加权亮度计算（通常为 `0.2126*R + 0.7152*G + 0.0722*B`），结果反映人眼对颜色亮度的感知。
  - **向量到浮点**（`VF`）使用 `average`，即简单的三分量算术平均 `(x+y+z)/3`，适合几何量的标量化。
- **标量到向量的扩展**：Float 和 Int 转换为 Vector 时，值被复制到所有三个分量 `(f, f, f)`，产生一个"灰色"向量。
- **整数转换的精度**：Float 到 Int 使用 `float_to_int`（可能包含四舍五入或截断逻辑），而 Color/Vector 到 Int 使用 C 风格的 `(int)` 强制转换（截断）。
- **KernelGlobals 参数**：虽然大多数转换不需要内核全局数据，但 `linear_rgb_to_gray` 需要访问颜色空间配置，因此函数签名中包含 `kg` 参数。

## 关联文件

- `kernel/svm/svm.h`：SVM 主调度入口
- `kernel/svm/util.h`：栈操作工具
- `kernel/util/colorspace.h`：提供 `linear_rgb_to_gray` 颜色空间转换函数
