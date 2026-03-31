# node_mapping.osl - 映射着色器

## 概述

该着色器对输入向量（点/纹理坐标/向量/法线）执行仿射变换，包括缩放、旋转和平移操作。支持四种映射模式，适用于不同类型的空间数据变换。属于 Cycles 渲染器开放着色语言（OSL）矢量工具节点。

## 着色器签名

```osl
shader node_mapping(
    string mapping_type = "point",
    point VectorIn = point(0.0, 0.0, 0.0),
    point Location = point(0.0, 0.0, 0.0),
    point Rotation = point(0.0, 0.0, 0.0),
    point Scale = point(1.0, 1.0, 1.0),
    output point VectorOut = point(0.0, 0.0, 0.0))
```

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| mapping_type | string | "point" | 映射类型，支持 "point"、"texture"、"vector"、"normal" |
| VectorIn | point | (0, 0, 0) | 输入向量/坐标 |
| Location | point | (0, 0, 0) | 平移量 |
| Rotation | point | (0, 0, 0) | 旋转角度（弧度），分别对应绕 X、Y、Z 轴旋转 |
| Scale | point | (1, 1, 1) | 缩放因子 |

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| VectorOut | point | 变换后的输出向量/坐标 |

## 辅助函数

### `safe_divide(point a, point b)`
安全的逐分量除法。当除数分量为零时返回零，避免除零错误。

### `euler_to_mat(point euler)`
将欧拉角（XYZ 顺序）转换为 3x3 旋转矩阵。使用标准的三角函数分解构建旋转矩阵。

## 实现逻辑

根据 `mapping_type` 执行不同的变换策略：

1. **"point"（点映射）**：先缩放，再旋转，最后平移。公式：`R * (V * S) + L`
2. **"texture"（纹理映射）**：点映射的逆操作。先平移（减去偏移），再用旋转矩阵的转置进行反向旋转，最后安全除以缩放因子。公式：`R^T * (V - L) / S`
3. **"vector"（向量映射）**：先缩放再旋转，但不执行平移（向量没有位置概念）。公式：`R * (V * S)`
4. **"normal"（法线映射）**：先安全除以缩放（法线的缩放是逆缩放），再旋转，最后归一化。公式：`normalize(R * (V / S))`

## 对应 SVM 节点

对应 SVM 内核中的 `NODE_MAPPING` 节点，类型为 `SHADER_NODE_MAPPING`。
