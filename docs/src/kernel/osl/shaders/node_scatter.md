# node_scatter.h - 体积散射辅助头文件

## 概述

该头文件提供了体积(Volume)散射相位函数的选择和构建逻辑，被体积散射着色器（`node_scatter_volume.osl`）引用。包含 Mie 散射的拟合参数计算以及统一的散射闭包选择函数。支持五种相位函数模型。

## 文件类型

C/开放着色语言(OSL) 头文件（.h），非独立着色器文件。

## 数据结构

### MieParameters

```osl
struct MieParameters {
    float g_HG;      // Henyey-Greenstein 各向异性参数
    float g_D;        // Draine 各向异性参数
    float alpha;      // Draine alpha 参数
    float mixture;    // HG 与 Draine 的混合比例
};
```

用于存储 Mie 散射拟合参数的结构体。Mie 散射通过 Henyey-Greenstein 和 Draine 相位函数的加权混合来近似。

## 函数列表

### phase_mie_fitted_parameters

```osl
MieParameters phase_mie_fitted_parameters(float Diameter)
```

根据粒子直径计算 Mie 散射的拟合参数。

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Diameter | float | 粒子直径（微米），最小值为 0 |
| 返回值 | MieParameters | 拟合的 Mie 散射参数 |

**实现逻辑**：

根据粒子直径的不同范围使用不同的拟合公式：

| 直径范围 | 对应公式 | 说明 |
|----------|----------|------|
| d <= 0.1 | 公式 (11-14) | 极小粒子，接近 Rayleigh 散射 |
| 0.1 < d < 1.5 | 公式 (15-18) | 中小粒子，使用对数空间拟合 |
| 1.5 <= d < 5.0 | 公式 (19-22) | 中等粒子 |
| d >= 5.0 | 公式 (7-10) | 大粒子，使用指数拟合 |

拟合公式来自学术文献，用于在实时渲染中高效近似 Mie 散射理论的精确解。

### scatter

```osl
closure color scatter(string phase, float Anisotropy, float IOR,
                      float Backscatter, float Alpha, float Diameter)
```

根据相位函数类型创建对应的散射闭包。

| 参数名 | 类型 | 说明 |
|--------|------|------|
| phase | string | 相位函数类型名称 |
| Anisotropy | float | 散射各向异性参数 |
| IOR | float | 粒子折射率（Fournier-Forand 用） |
| Backscatter | float | 后向散射比例（Fournier-Forand 用） |
| Alpha | float | Draine alpha 参数 |
| Diameter | float | 粒子直径（Mie 用） |
| 返回值 | closure color | 散射闭包 |

**实现逻辑**：

根据 `phase` 参数选择不同的散射闭包：

| 相位函数 | 闭包 | 参数 | 适用场景 |
|----------|------|------|----------|
| `"Fournier-Forand"` | `fournier_forand(Backscatter, IOR)` | 后向散射比例和粒子折射率 | 水下粒子散射 |
| `"Draine"` | `draine(Anisotropy, Alpha)` | 各向异性和 alpha | 天文尘埃散射 |
| `"Rayleigh"` | `rayleigh()` | 无参数 | 小粒子散射（如天空） |
| `"Mie"` | `mix(henyey_greenstein, draine, mixture)` | 通过拟合参数混合 | 球形水滴散射 |
| 默认（Henyey-Greenstein） | `henyey_greenstein(Anisotropy)` | 各向异性 | 通用体积散射 |

Mie 模型通过 `phase_mie_fitted_parameters` 获取拟合参数后，将 Henyey-Greenstein 和 Draine 闭包按混合比例进行混合，近似真实的 Mie 散射分布。

## 被引用的着色器

- `node_scatter_volume.osl`
