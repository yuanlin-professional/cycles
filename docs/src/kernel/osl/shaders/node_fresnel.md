# node_fresnel.h - 菲涅耳计算辅助头文件

## 概述

该头文件提供了菲涅耳反射率计算的辅助函数，被多个双向散射分布函数(BSDF)着色器引用（如玻璃、光泽、金属、光泽绒面、原理化着色器等）。包含电介质菲涅耳、导体菲涅耳以及折射率与法线入射反射率之间的相互转换函数。

代码源自 Open Shading Language，并经 Blender Foundation 适配。

## 文件类型

C/开放着色语言(OSL) 头文件（.h），非独立着色器文件。

## 函数列表

### fresnel_dielectric_cos

```osl
float fresnel_dielectric_cos(float cosi, float eta)
```

计算电介质材质的菲涅耳反射率。

| 参数名 | 类型 | 说明 |
|--------|------|------|
| cosi | float | 入射角余弦值 |
| eta | float | 折射率（外部介质与内部介质之比） |
| 返回值 | float | 菲涅耳反射率，范围 [0, 1] |

**实现逻辑**：
1. 计算 `g = eta^2 - 1 + cos^2(i)`。
2. 若 `g > 0`（存在折射光路），使用精确菲涅耳公式计算 s 偏振和 p 偏振的平均反射率。
3. 若 `g <= 0`（全内反射），返回 1.0。

### fresnel_conductor

```osl
color fresnel_conductor(float cosi, color eta, color k)
```

计算导体（金属）材质的菲涅耳反射率，支持每通道独立的复折射率。

| 参数名 | 类型 | 说明 |
|--------|------|------|
| cosi | float | 入射角余弦值 |
| eta | color | 每通道折射率（实部） |
| k | color | 每通道消光系数（虚部） |
| 返回值 | color | 每通道菲涅耳反射率 |

**实现逻辑**：
- 分别计算平行偏振（Rparl2）和垂直偏振（Rperp2）的反射率。
- 返回两者的平均值。

### F0_from_ior

```osl
float F0_from_ior(float eta)
```

从折射率计算法线入射时的反射率（F0）。

| 参数名 | 类型 | 说明 |
|--------|------|------|
| eta | float | 折射率 |
| 返回值 | float | 法线入射反射率 F0 |

**实现逻辑**：使用公式 `F0 = ((eta - 1) / (eta + 1))^2`。

### ior_from_F0

```osl
float ior_from_F0(float f0)
```

从法线入射反射率反算折射率。

| 参数名 | 类型 | 说明 |
|--------|------|------|
| f0 | float | 法线入射反射率，钳制到 [0, 0.99] |
| 返回值 | float | 对应的折射率 |

**实现逻辑**：使用公式 `IOR = (1 + sqrt(F0)) / (1 - sqrt(F0))`，为 `F0_from_ior` 的逆运算。F0 钳制到 0.99 以避免除零。

## 被引用的着色器

- `node_glossy_bsdf.osl`
- `node_glass_bsdf.osl`
- `node_metallic_bsdf.osl`
- `node_sheen_bsdf.osl`
- `node_principled_bsdf.osl`
