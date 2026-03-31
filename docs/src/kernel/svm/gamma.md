# gamma.h - 伽马校正节点

## 概述

`gamma.h` 实现了 Cycles SVM 中的伽马（Gamma）校正节点，对应 Blender 着色器编辑器中的"伽马（Gamma）"节点。该节点对输入颜色的每个通道应用幂函数 `pow(color, gamma)` 进行伽马变换，常用于颜色空间校正和色调调整。

## 核心函数

### `svm_node_gamma`

```c
ccl_device_noinline void svm_node_gamma(ccl_private float *stack,
                                        const uint in_gamma,
                                        const uint in_color,
                                        const uint out_color)
```

- **功能**: 伽马校正节点的 SVM 执行入口。
- **参数**:
  - `stack`: SVM 栈指针。
  - `in_gamma`: 伽马值的栈偏移量。
  - `in_color`: 输入颜色的栈偏移量。
  - `out_color`: 输出颜色的栈偏移量。
- **流程**: 从栈加载颜色和伽马值，调用 `svm_math_gamma_color()` 进行逐通道伽马校正，将结果存回栈。

## 依赖关系

- **内部头文件**:
  - `kernel/svm/math_util.h` — 提供 `svm_math_gamma_color()` 实际伽马校正实现
  - `kernel/svm/util.h` — 栈操作工具
- **被引用**:
  - `kernel/svm/svm.h` — SVM 主调度器，在 `NODE_GAMMA` 指令分支中调用

## 实现细节 / 关键算法

### 伽马校正算法（`svm_math_gamma_color` 在 math_util.h 中）

- **gamma = 0**: 直接返回白色 `(1, 1, 1)`，这是 `x^0 = 1` 的数学约定。
- **正值通道**: 对 `color.x/y/z > 0` 的通道执行 `powf(channel, gamma)`。
- **非正值通道**: 保持不变（跳过 `pow` 运算），避免对负数或零取幂产生 NaN。

### 输出有效性检查

使用 `stack_valid(out_color)` 检查输出偏移是否有效后再写入，这是 SVM 节点的标准模式，允许输出被优化掉（未连接时不写入）。

## 关联文件

- `kernel/svm/math_util.h` — `svm_math_gamma_color()` 的定义位置
- `kernel/svm/svm.h` — SVM 指令调度器
- `kernel/svm/brightness.h` — 另一种颜色调整方式（亮度/对比度）
