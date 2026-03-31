# node_bump.osl - 凹凸贴图着色器

## 概述

该着色器基于 Morten S. Mikkelsen 于 2010 年发表的论文 "Bump Mapping Unparameterized Surfaces on the GPU" 实现凹凸贴图。通过对三个采样点（中心、X 方向、Y 方向）的高度差计算扰动法线，从而在不改变几何体的情况下产生表面凹凸细节。此着色器声明为 `surface` 类型。属于 Cycles 渲染器开放着色语言（OSL）矢量工具节点。

## 着色器签名

```osl
surface node_bump(
    int invert = 0,
    int use_object_space = 0,
    normal NormalIn = N,
    float Strength = 0.1,
    float Distance = 1.0,
    float FilterWidth = BUMP_FILTER_WIDTH,
    float SampleCenter = 0.0,
    float SampleX = 0.0,
    float SampleY = 0.0,
    output normal NormalOut = N)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| invert | int | 0 | 是否反转凹凸方向（0 为不反转，非零为反转） |
| use_object_space | int | 0 | 是否在物体空间中计算（0 为世界空间，非零为物体空间） |
| NormalIn | normal | N | 输入法线，默认使用着色点法线 |
| Strength | float | 0.1 | 凹凸强度，范围 [0, +inf)，自动钳制为非负值 |
| Distance | float | 1.0 | 凹凸距离缩放因子 |
| FilterWidth | float | BUMP_FILTER_WIDTH (0.1) | 滤波宽度，用于控制法线扰动的平滑程度 |
| SampleCenter | float | 0.0 | 中心点的高度采样值 |
| SampleX | float | 0.0 | X 方向偏移点的高度采样值 |
| SampleY | float | 0.0 | Y 方向偏移点的高度采样值 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| NormalOut | normal | 经凹凸扰动后的输出法线 |

## 实现逻辑

1. **坐标准备**：获取当前着色点位置 `P` 和输入法线。若启用物体空间，将两者变换到物体空间。
2. **表面切线计算**：使用 `Dx(P)` 和 `Dy(P)` 获取屏幕空间的表面微分，并计算两个辅助向量：
   - `Rx = cross(dPdy, Normal)`
   - `Ry = cross(Normal, dPdx)`
3. **表面梯度计算**：基于高度差 `(SampleX - SampleCenter)` 和 `(SampleY - SampleCenter)` 与辅助向量计算表面梯度。
4. **法线扰动**：结合滤波宽度、行列式绝对值和距离因子计算扰动法线：
   - `NormalOut = normalize(FilterWidth * |det| * N - dist * sign(det) * surfgrad)`
5. **强度混合**：将扰动法线与原始法线按 `Strength` 参数进行线性混合。
6. **空间变换**：若启用物体空间，将结果法线变换回世界空间并归一化。

## 对应 SVM 节点

对应 SVM 内核中的 `NODE_SET_BUMP` 节点，类型为 `SHADER_NODE_BUMP`。
