# node_noise.h - 噪声函数库头文件

## 概述

该头文件是开放着色语言(OSL)着色器系统的核心噪声函数库。提供安全的 Perlin 噪声封装以及五种分形噪声算法的模板化实现。所有噪声函数支持 float、vector2、vector3（point）和 vector4 四种输入类型，被 `node_noise_texture.osl` 和 `node_wave_texture.osl` 等着色器引用。

## 文件依赖

```
node_noise.h
  ├── vector2.h    (vector2 类型定义)
  └── vector4.h    (vector4 类型定义)
```

## 核心函数

### safe_noise 系列 - 安全 Perlin 噪声

| 函数签名 | 说明 |
|----------|------|
| `float safe_noise(float co)` | 1D Perlin 噪声，输出 [0, 1] |
| `float safe_noise(vector2 co)` | 2D Perlin 噪声，输出 [0, 1] |
| `float safe_noise(vector3 co)` | 3D Perlin 噪声，输出 [0, 1] |
| `float safe_noise(vector4 co)` | 4D Perlin 噪声，输出 [0, 1] |

**安全处理**：对坐标执行 `fmod(co, 100000.0)` 取模运算，防止浮点精度在大坐标值下产生伪影。当坐标绝对值超过 1000000.0 时添加 0.5 的精度修正。

### safe_snoise 系列 - 安全有符号 Perlin 噪声

| 函数签名 | 说明 |
|----------|------|
| `float safe_snoise(float co)` | 1D 有符号噪声，输出 [-1, 1] |
| `float safe_snoise(vector2 co)` | 2D 有符号噪声，输出 [-1, 1] |
| `float safe_snoise(vector3 co)` | 3D 有符号噪声，输出 [-1, 1] |
| `float safe_snoise(vector4 co)` | 4D 有符号噪声，输出 [-1, 1] |

使用 OSL 内置 `noise("snoise", ...)` 函数，同样具有坐标安全处理。

## 分形噪声算法

以下算法通过宏模板（`NOISE_FBM`、`NOISE_MULTI_FRACTAL` 等）实例化为 float/vector2/vector3/vector4 四个版本。

### noise_fbm - 分形布朗运动

```
float noise_fbm(T co, float detail, float roughness, float lacunarity, int use_normalize)
```

标准 fBM 分形噪声。每层迭代频率乘以 `lacunarity`，振幅乘以 `roughness`。支持小数 detail 层级的平滑过渡。当 `use_normalize` 为真时，将输出映射到 [0, 1] 范围（`0.5 * sum / maxamp + 0.5`）。

### noise_multi_fractal - 多重分形

```
float noise_multi_fractal(T co, float detail, float roughness, float lacunarity)
```

多重分形噪声，各层级通过乘法组合（而非加法），初始值为 1.0。产生更具对比度的分形图案。

### noise_hetero_terrain - 异质地形

```
float noise_hetero_terrain(T co, float detail, float roughness, float lacunarity, float offset)
```

异质地形噪声。第一层为 `offset + snoise(p)`，后续层级的贡献由前一层级的值调制（`increment * value`），使低处更平坦、高处更粗糙。

### noise_hybrid_multi_fractal - 混合多重分形

```
float noise_hybrid_multi_fractal(T co, float detail, float roughness, float lacunarity, float offset, float gain)
```

混合多重分形噪声，结合加法和乘法分形。权重由 `gain * signal` 自适应调节，当权重低于 0.001 时提前退出，适合生成具有平坦区域和粗糙峰顶的地形。

### noise_ridged_multi_fractal - 脊状多重分形

```
float noise_ridged_multi_fractal(T co, float detail, float roughness, float lacunarity, float offset, float gain)
```

脊状多重分形噪声。对噪声取绝对值后从 offset 中减去（`offset - |snoise(p)|`），再平方，产生尖锐的脊线特征。适合模拟山脉脊线和裂纹。

## 被引用文件

- `node_noise_texture.osl` - 噪声纹理着色器
- `node_wave_texture.osl` - 波纹纹理着色器
