# node_camera.osl - 相机信息着色器

## 概述

相机信息着色器（Camera Data Node）用于获取当前着色点相对于相机的空间信息。该着色器是开放着色语言（OSL）中的输入类节点，提供从着色点到相机的视图向量、Z 轴深度和距离信息。

## 着色器签名

```osl
shader node_camera(
    output vector ViewVector = vector(0.0, 0.0, 0.0),
    output float ViewZDepth = 0.0,
    output float ViewDistance = 0.0
)
```

## 输入参数

无输入参数。

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| ViewVector | vector | 从着色点指向相机的归一化方向向量（相机空间） |
| ViewZDepth | float | 着色点在相机空间中的 Z 轴深度值 |
| ViewDistance | float | 着色点到相机的欧几里得距离 |

## 实现逻辑

1. **坐标变换**：将着色点世界空间位置 `P` 变换到相机空间，得到相机空间坐标向量。

2. **Z 轴深度**：取相机空间坐标的 Z 分量作为 `ViewZDepth`，表示沿相机视轴方向的深度。

3. **距离计算**：对相机空间坐标向量取模长，得到着色点到相机的实际距离 `ViewDistance`。

4. **方向归一化**：对相机空间坐标向量进行归一化，得到最终的 `ViewVector` 输出。

## 对应 SVM 节点

对应 Cycles SVM（着色器虚拟机）中的 `NODE_CAMERA` 节点。
