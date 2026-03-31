# node_fractal_voronoi.h - 分形 Voronoi 函数库头文件

## 概述

该头文件是开放着色语言(OSL)着色器系统中 Voronoi 纹理的分形叠加层。在 `node_voronoi.h` 提供的基础 Voronoi 函数之上，实现类似 fBM 的多层级分形叠加逻辑。支持 F1/F2/平滑 F1 特征的分形化以及到边缘距离的分形化，为 Voronoi 纹理提供更丰富的细节控制。

## 文件依赖

```
node_fractal_voronoi.h
  ├── node_voronoi.h   (基础 Voronoi 函数)
  ├── stdcycles.h       (Cycles 标准库)
  ├── vector2.h         (vector2 类型)
  └── vector4.h         (vector4 类型)
```

## 核心宏模板

### FRACTAL_VORONOI_X_FX - 分形 Voronoi 特征

```osl
VoronoiOutput fractal_voronoi_x_fx(VoronoiParams params, T coord)
```

**支持类型**：float、vector2、vector3、vector4

**功能**：对 Voronoi F1/F2/smooth_f1 特征执行多层级分形叠加。

**实现逻辑**：

1. **迭代叠加**：循环 `ceil(detail)` 次，每次计算一个八度（octave）的 Voronoi 值。
2. **特征选择**：根据 `params.feature` 调用 `voronoi_f1`、`voronoi_f2` 或 `voronoi_smooth_f1`。
3. **振幅衰减**：每层迭代 `amplitude *= roughness`，`scale *= lacunarity`。
4. **累加方式**：
   - 距离：`Output.Distance += octave.Distance * amplitude`
   - 颜色：`Output.Color += octave.Color * amplitude`
   - 位置：`mix(Output.Position, octave.Position / scale, amplitude)`（线性插值）
5. **小数层级**：对非整数 detail 值，使用 `mix` 在最后一层进行平滑过渡。
6. **零输入优化**：当 `detail == 0` 或 `roughness == 0` 时，跳过分形直接返回单层结果。
7. **归一化**：当 `params.normalize` 启用时，距离除以 `max_amplitude * max_distance`，颜色除以 `max_amplitude`。
8. **位置还原**：最终位置除以 `params.scale` 还原到原始坐标空间。

### FRACTAL_VORONOI_DISTANCE_TO_EDGE_FUNCTION - 分形边缘距离

```osl
float fractal_voronoi_distance_to_edge(VoronoiParams params, T coord)
```

**支持类型**：float、vector2、vector3、vector4

**功能**：对 Voronoi 到边缘距离执行多层级分形叠加。

**实现逻辑**：

1. **迭代叠加**：与 x_fx 类似的循环结构，但仅计算标量距离。
2. **距离混合**：使用 `mix(distance, min(distance, octave_distance / scale), amplitude)` 进行混合，取较小值以保留边缘特征。
3. **最大振幅追踪**：`max_amplitude = mix(max_amplitude, max_distance / scale, amplitude)`，随层级缩小参考距离。
4. **归一化**：当启用时，距离除以 `max_amplitude`。

## 实例化

```osl
// 1D 分形 Voronoi
FRACTAL_VORONOI_X_FX(float)
FRACTAL_VORONOI_DISTANCE_TO_EDGE_FUNCTION(float)

// 2D 分形 Voronoi
FRACTAL_VORONOI_X_FX(vector2)
FRACTAL_VORONOI_DISTANCE_TO_EDGE_FUNCTION(vector2)

// 3D 分形 Voronoi
FRACTAL_VORONOI_X_FX(vector3)
FRACTAL_VORONOI_DISTANCE_TO_EDGE_FUNCTION(vector3)

// 4D 分形 Voronoi
FRACTAL_VORONOI_X_FX(vector4)
FRACTAL_VORONOI_DISTANCE_TO_EDGE_FUNCTION(vector4)
```

## 被引用文件

- `node_voronoi_texture.osl` - Voronoi 纹理着色器
