# node_gradient_texture.osl - 渐变纹理着色器

## 概述

该开放着色语言(OSL)着色器节点生成多种类型的渐变纹理图案。支持线性、二次、缓动、对角线、径向、球形和二次球形共七种渐变模式，输出值始终限制在 [0, 1] 范围内，常用于创建遮罩、颜色渐变和程序化材质效果。

## 着色器签名

```osl
shader node_gradient_texture(
    int use_mapping = 0,
    matrix mapping = matrix(0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0),
    string gradient_type = "linear",
    point Vector = P,
    output float Fac = 0.0,
    output color Color = 0.0)
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| use_mapping | int | 0 | 是否启用纹理坐标映射变换 |
| mapping | matrix | 零矩阵 | 纹理坐标变换矩阵 |
| gradient_type | string | "linear" | 渐变类型（见下方说明） |
| Vector | point | P | 输入纹理坐标 |

**gradient_type 可选值：**
- `"linear"` - 线性渐变，沿 X 轴线性变化
- `"quadratic"` - 二次渐变，沿 X 轴二次方变化（仅正半轴）
- `"easing"` - 缓动渐变，使用三次平滑曲线 `3t^2 - 2t^3`
- `"diagonal"` - 对角线渐变，沿 `(x + y) / 2` 方向
- `"radial"` - 径向渐变，基于 X-Y 平面角度
- `"spherical"` - 球形渐变，基于到原点的距离（线性衰减）
- `"quadratic_sphere"` - 二次球形渐变，基于到原点距离的二次方衰减

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Fac | float | 渐变值，范围 [0, 1] |
| Color | color | 灰度颜色输出（RGB 三通道均为 Fac 值） |

## 实现逻辑

1. **坐标变换**：可选应用 mapping 矩阵变换输入坐标。
2. **渐变计算**：根据 `gradient_type` 选择不同计算公式：
   - `linear`：直接取 x 分量
   - `quadratic`：`max(x, 0)^2`
   - `easing`：`3r^2 - 2r^3`（r 限制在 [0, 1]）
   - `diagonal`：`(x + y) * 0.5`
   - `radial`：`atan2(y, x) / (2*PI) + 0.5`
   - `spherical` / `quadratic_sphere`：基于 `1 - sqrt(x^2 + y^2 + z^2)` 的线性或二次衰减
3. **输出限制**：最终结果使用 `clamp(result, 0.0, 1.0)` 限制在有效范围内。

## 对应 SVM 节点

对应 `svm_gradient` 相关实现（`src/kernel/svm/gradient.h`）。
