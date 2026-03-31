# brick.h - 砖块纹理 SVM 节点

## 概述
`brick.h` 实现了 Cycles 的砖块纹理节点(`NODE_TEX_BRICK`)，生成程序化的砖墙图案。该纹理模拟真实砖墙的排列模式，支持砖块宽度、行高、偏移频率、挤压效果、砂浆(mortar)宽度和平滑度等参数，可输出砖块颜色（支持两色随机混合）和砂浆因子。

该节点是一个纯几何计算的纹理，不依赖噪声函数，仅使用简单的整数哈希生成砖块色调变化。

## 核心函数

### `brick_noise(n)`
- **功能**: 快速整数噪声函数，生成 [0, 0.5] 范围的伪随机值
- **算法**: 基于整数位运算的哈希：`(n * (n * n * 60493 + 19990303) + 1376312589) & 0x7fffffff`
- **用途**: 为每块砖生成唯一的色调(tint)值

### `svm_brick(p, mortar_size, mortar_smooth, bias, brick_width, row_height, offset_amount, offset_frequency, squash_amount, squash_frequency)`
- **功能**: 核心砖块纹理计算
- **返回**: `float2(tint, mortar)` —— 砖块色调和砂浆因子
- **算法**:
  1. 根据 `row_height` 计算行号 `rownum`
  2. 根据 `squash_frequency` 和 `squash_amount` 对特定行应用挤压（改变砖块宽度）
  3. 根据 `offset_frequency` 和 `offset_amount` 对特定行应用水平偏移（交错排列）
  4. 计算砖块编号 `bricknum` 和砖内局部坐标 `(x, y)`
  5. 通过 `brick_noise` 使用行号和砖号生成色调值
  6. 计算到砖块边缘的最小距离 `min_dist`，判断是否在砂浆区域
  7. 使用 `smoothstepf` 对砂浆边缘进行平滑处理

### `svm_node_tex_brick(kg, stack, node, offset)`
- **功能**: SVM 节点入口函数
- **输入参数**: 坐标(co)、颜色1(color1)、颜色2(color2)、砂浆颜色(mortar)、缩放(scale)、砂浆大小(mortar_size)、砂浆平滑度(mortar_smooth)、偏差(bias)、砖宽(brick_width)、行高(row_height)
- **RNA 属性**: 偏移频率(offset_frequency)、挤压频率(squash_frequency)、偏移量(offset_amount)、挤压量(squash_amount)
- **输出**:
  - **颜色**: 砖块区域为 `color1` 和 `color2` 的色调混合，砂浆区域为砂浆颜色
  - **因子**: 砂浆因子 `f`（0 = 砖块，1 = 砂浆）

## 依赖关系
- **内部头文件**:
  - `kernel/svm/util.h` - 栈操作和节点读取
- **被引用**:
  - `kernel/svm/svm.h` - 主解释器通过 `NODE_TEX_BRICK` case 调用

## 实现细节

### 砖块偏移与挤压
- **偏移(Offset)**: 每隔 `offset_frequency` 行，砖块水平偏移 `brick_width * offset_amount`。典型设置为 `frequency=2, amount=0.5` 产生标准交错砖墙
- **挤压(Squash)**: 每隔 `squash_frequency` 行，砖块宽度乘以 `squash_amount`，产生宽窄交替的效果

### 砂浆平滑处理
砂浆区域判断基于到砖块四条边的最小距离 `min_dist`：
- `min_dist >= mortar_size`: 在砖块内部，mortar = 0
- `mortar_smooth == 0`: 硬边界，mortar = 1
- 否则: `mortar = smoothstep(1 - min_dist/mortar_size) / mortar_smooth)`

### 颜色混合
砖块颜色通过色调值在 `color1` 和 `color2` 之间插值：
```cpp
color1 = (1 - tint) * color1 + tint * color2;
```
最终输出通过砂浆因子 `f` 在砖块颜色和砂浆颜色之间混合：
```cpp
output = color1 * (1 - f) + mortar * f;
```

### 参数读取
该节点的参数较多，需要读取 3 个额外的 `uint4` 节点（`node2`, `node3`, `node4`），总共使用 4 个 `uint4`（16 个 uint）存储所有栈偏移量、默认值和 RNA 属性。

## 关联文件
- `kernel/svm/types.h` - 无特定枚举（砖块纹理使用原始整数参数）
- `kernel/svm/svm.h` - SVM 主解释器
- `kernel/svm/checker.h` - 类似的几何图案纹理
