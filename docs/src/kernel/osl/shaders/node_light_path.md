# node_light_path.osl - 光线路径着色器

## 概述

光线路径着色器（Light Path Node）用于获取当前光线的类型和路径深度信息。该着色器是开放着色语言（OSL）中的输入类节点，在基于路径追踪的渲染器中，可以根据光线类型对材质进行条件分支，实现不同渲染通道下的差异化表现。

## 着色器签名

```osl
shader node_light_path(
    output float IsCameraRay = 0.0,
    output float IsShadowRay = 0.0,
    output float IsDiffuseRay = 0.0,
    output float IsGlossyRay = 0.0,
    output float IsSingularRay = 0.0,
    output float IsReflectionRay = 0.0,
    output float IsTransmissionRay = 0.0,
    output float IsVolumeScatterRay = 0.0,
    output float RayLength = 0.0,
    output float RayDepth = 0.0,
    output float DiffuseDepth = 0.0,
    output float GlossyDepth = 0.0,
    output float TransparentDepth = 0.0,
    output float TransmissionDepth = 0.0,
    output float PortalDepth = 0.0
)
```

## 输入参数

无输入参数。

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| IsCameraRay | float | 是否为相机光线（直接从相机发出的主光线），1.0 为是 |
| IsShadowRay | float | 是否为阴影光线，1.0 为是 |
| IsDiffuseRay | float | 是否为漫射光线，1.0 为是 |
| IsGlossyRay | float | 是否为光泽光线，1.0 为是 |
| IsSingularRay | float | 是否为奇异光线（如完美镜面反射/折射），1.0 为是 |
| IsReflectionRay | float | 是否为反射光线，1.0 为是 |
| IsTransmissionRay | float | 是否为透射光线（折射），1.0 为是 |
| IsVolumeScatterRay | float | 是否为体积散射光线，1.0 为是 |
| RayLength | float | 当前光线的长度 |
| RayDepth | float | 当前光线的总弹射深度 |
| DiffuseDepth | float | 漫射弹射深度 |
| GlossyDepth | float | 光泽弹射深度 |
| TransparentDepth | float | 透明弹射深度 |
| TransmissionDepth | float | 透射弹射深度 |
| PortalDepth | float | 门户弹射深度 |

## 实现逻辑

1. **光线类型检测**：使用开放着色语言（OSL）内建函数 `raytype()` 检测当前光线类型，传入对应的类型字符串（`"camera"`、`"shadow"`、`"diffuse"`、`"glossy"`、`"singular"`、`"reflection"`、`"refraction"`、`"volume_scatter"`），返回 1.0 或 0.0。

2. **光线长度**：通过 `getattribute("path:ray_length", ...)` 获取当前光线的路径长度。

3. **深度信息**：分别通过 `getattribute` 获取各类弹射深度属性（`path:ray_depth`、`path:diffuse_depth`、`path:glossy_depth`、`path:transparent_depth`、`path:transmission_depth`、`path:portal_depth`），获取值为整数后转换为浮点数输出。

## 对应 SVM 节点

对应 Cycles SVM（着色器虚拟机）中的 `NODE_LIGHT_PATH` 节点。各光线类型和深度信息在 SVM 中通过不同的子类型枚举值区分。
