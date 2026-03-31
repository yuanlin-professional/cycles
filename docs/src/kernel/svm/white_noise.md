# white_noise.h - 白噪声纹理节点

## 概述

`white_noise.h` 实现了 Cycles SVM 中的白噪声纹理（White Noise Texture）节点，对应 Blender 着色器编辑器中的"白噪声纹理（White Noise Texture）"节点。该节点基于哈希函数生成确定性的伪随机数，支持 1D 到 4D 维度输入，同时输出随机标量值和随机颜色。常用于程序化材质中的随机化效果。

## 核心函数

### `svm_node_tex_white_noise`

```c
ccl_device_noinline void svm_node_tex_white_noise(ccl_private float *stack,
                                                   const uint dimensions,
                                                   const uint inputs_stack_offsets,
                                                   const uint outputs_stack_offsets)
```

- **功能**: 白噪声纹理节点的 SVM 执行入口。
- **参数**:
  - `stack`: SVM 栈指针。
  - `dimensions`: 噪声维度（1、2、3 或 4）。
  - `inputs_stack_offsets`: 打包的输入偏移量（向量 vector 和标量 w）。
  - `outputs_stack_offsets`: 打包的输出偏移量（标量 value 和颜色 color）。
- **输入**:
  - `vector`: float3 向量输入（用于 2D/3D/4D）。
  - `w`: 浮点标量输入（用于 1D/4D）。
- **输出**:
  - `value`: [0, 1] 范围内的伪随机浮点值。
  - `color`: 三通道独立的伪随机颜色。

## 依赖关系

- **内部头文件**:
  - `kernel/svm/util.h` — 栈操作和节点解包工具
  - `util/hash.h` — 提供各维度的哈希函数（`hash_float_to_float`、`hash_float_to_float3` 等）
- **被引用**:
  - `kernel/svm/svm.h` — SVM 主调度器，在 `NODE_TEX_WHITE_NOISE` 指令分支中调用

## 实现细节 / 关键算法

### 维度与哈希函数映射

| 维度 | 标量输出哈希函数 | 颜色输出哈希函数 | 输入 |
|------|----------------|----------------|------|
| 1D | `hash_float_to_float(w)` | `hash_float_to_float3(w)` | 仅标量 w |
| 2D | `hash_float2_to_float(x, y)` | `hash_float2_to_float3(x, y)` | 向量的 x、y 分量 |
| 3D | `hash_float3_to_float(vector)` | `hash_float3_to_float3(vector)` | 完整的三维向量 |
| 4D | `hash_float4_to_float(vector, w)` | `hash_float4_to_float3(vector, w)` | 三维向量 + 标量 w |

### 按需计算

通过 `stack_valid()` 检查输出偏移量是否有效，只有在对应输出被连接时才执行哈希计算。颜色输出和标量输出独立检查，未连接的输出不会浪费计算资源。

### 白噪声特性

与 Perlin 噪声等空间相关噪声不同，白噪声的特点是：
- **空间不相关**: 相邻坐标的输出完全无关联，没有平滑过渡。
- **确定性**: 相同输入总是产生相同输出（基于哈希函数）。
- **均匀分布**: 输出值在 [0, 1] 范围内近似均匀分布。

### 错误处理

`default` 分支中使用 `kernel_assert(0)` 标记不应到达的代码路径，颜色输出设为品红色 `(1, 0, 1)`（常见的错误指示色），标量输出设为 0。

## 关联文件

- `util/hash.h` — 底层哈希函数实现
- `kernel/svm/svm.h` — SVM 指令调度器
- `kernel/svm/noise.h` — 其他噪声纹理节点（Perlin 噪声等）
