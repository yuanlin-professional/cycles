# node_particle_info.osl - 粒子信息着色器

## 概述

粒子信息着色器（Particle Info Node）用于获取当前粒子系统中各粒子的属性信息。该着色器是开放着色语言（OSL）中的输入类节点，仅在物体由粒子系统实例化时有效，可获取粒子的索引、年龄、生命周期、位置、大小、速度和角速度等信息。

## 着色器签名

```osl
shader node_particle_info(
    output float Index = 0.0,
    output float Random = 0.0,
    output float Age = 0.0,
    output float Lifetime = 0.0,
    output point Location = point(0.0, 0.0, 0.0),
    output float Size = 0.0,
    output vector Velocity = point(0.0, 0.0, 0.0),
    output vector AngularVelocity = point(0.0, 0.0, 0.0)
)
```

## 输入参数

无输入参数。

## 输出参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| Index | float | 粒子索引编号 |
| Random | float | 每个粒子的随机值，取值范围 [0, 1] |
| Age | float | 粒子当前年龄（已存活时间），单位为帧 |
| Lifetime | float | 粒子的总生命周期长度，单位为帧 |
| Location | point | 粒子的世界空间位置 |
| Size | float | 粒子的大小 |
| Velocity | vector | 粒子的速度向量 |
| AngularVelocity | vector | 粒子的角速度向量 |

## 实现逻辑

所有输出参数均通过 `getattribute` 函数从渲染器的粒子数据中获取：

- `"particle:index"` → 粒子索引
- `"particle:random"` → 粒子随机值
- `"particle:age"` → 粒子年龄
- `"particle:lifetime"` → 粒子生命周期
- `"particle:location"` → 粒子位置
- `"particle:size"` → 粒子大小
- `"particle:velocity"` → 粒子速度
- `"particle:angular_velocity"` → 粒子角速度

实现非常直接，无额外计算逻辑。

## 对应 SVM 节点

对应 Cycles SVM（着色器虚拟机）中的 `NODE_PARTICLE_INFO` 节点。
