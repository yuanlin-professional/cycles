# gradient.h - 渐变纹理 SVM 节点

## 概述
`gradient.h` 实现了 Cycles 的渐变纹理节点(`NODE_TEX_GRADIENT`)，生成多种类型的程序化渐变图案。支持七种渐变模式：线性、二次、缓动、对角线、径向、二次球面和球面，覆盖了从简单线性过渡到复杂径向衰减的多种需求。

该纹理计算非常轻量，仅依赖基本数学运算（无噪声采样），适合用作遮罩、混合因子或简单的空间变化着色。

## 核心函数

### `svm_gradient(p, type)`
- **功能**: 根据渐变类型计算 3D 空间中点 `p` 的渐变值
- **返回**: `float`，未截断的原始渐变值
- **渐变类型**:

| 类型 | 枚举值 | 算法 | 描述 |
|------|--------|------|------|
| 线性 | `NODE_BLEND_LINEAR` | `x` | 沿 X 轴线性渐变 |
| 二次 | `NODE_BLEND_QUADRATIC` | `max(x, 0)^2` | 沿 X 轴二次渐变（仅正向） |
| 缓动 | `NODE_BLEND_EASING` | `3t^2 - 2t^3` (Hermite) | X 轴上的平滑阶梯，输入钳制到 [0, 1] |
| 对角线 | `NODE_BLEND_DIAGONAL` | `(x + y) * 0.5` | XY 平面对角线渐变 |
| 径向 | `NODE_BLEND_RADIAL` | `atan2(y, x) / 2pi + 0.5` | 绕 Z 轴的角度渐变 |
| 二次球面 | `NODE_BLEND_QUADRATIC_SPHERE` | `r^2` | 从原点向外的二次球面衰减 |
| 球面 | `NODE_BLEND_SPHERICAL` | `r` | 从原点向外的线性球面衰减 |

其中球面类 `r = max(0.999999 - sqrt(x^2 + y^2 + z^2), 0)`，使用 0.999999 偏移确保单位向量上精确为零。

### `svm_node_tex_gradient(stack, node)`
- **功能**: SVM 节点入口函数
- **输入**: 坐标(co)、渐变类型(type)
- **处理**: 调用 `svm_gradient` 后通过 `saturatef` 截断到 [0, 1]
- **输出**:
  - **因子(Fac)**: 截断后的渐变值
  - **颜色**: `float3(f, f, f)` 灰度颜色

## 依赖关系
- **内部头文件**:
  - `kernel/svm/util.h` - 栈操作和节点读取
- **被引用**:
  - `kernel/svm/svm.h` - 主解释器通过 `NODE_TEX_GRADIENT` case 调用

## 实现细节

### 球面渐变的精度处理
球面类渐变使用 `0.999999 - sqrt(...)` 而非 `1.0 - sqrt(...)`，这是因为当输入点恰好位于单位球面上时（如归一化法线），`sqrt(x^2+y^2+z^2)` 可能因浮点精度略大于 1.0，导致结果出现微小负值。0.999999 的偏移确保这种情况下精确返回 0。

### 缓动渐变
缓动(Easing)模式使用 Hermite 插值多项式 `3t^2 - 2t^3`，这与 `smoothstep` 函数相同，在 t=0 和 t=1 处导数为零，产生平滑的 S 形过渡。

### 无额外节点数据
与棋盘格纹理类似，渐变纹理的所有参数（类型、坐标偏移、输出偏移）都打包在单个 `uint4 node` 中，无需读取额外数据节点。

### 输出截断
`saturatef(f)` 将结果钳制到 [0, 1] 范围。线性和对角线渐变可能产生超出此范围的值（取决于坐标），截断确保输出始终是有效的因子值。

## 关联文件
- `kernel/svm/types.h` - `NodeGradientType` 枚举（7 种渐变类型）
- `kernel/svm/checker.h` - 另一种简单程序化纹理
- `kernel/svm/svm.h` - SVM 主解释器
