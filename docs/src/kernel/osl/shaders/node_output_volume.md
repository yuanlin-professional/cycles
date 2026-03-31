# node_output_volume.osl - 体积输出着色器

## 概述

体积输出着色器（Volume Output Node）是开放着色语言（OSL）中的输出类节点，用于将计算好的体积闭包（Closure）赋值给渲染器的最终体积输出。它是体积材质节点树的终端节点，将所有体积着色计算结果传递给 Cycles 渲染器。

## 着色器签名

```osl
volume node_output_volume(
    closure color Volume = 0
)
```

注意：该着色器类型为 `volume`，而非通用 `shader`。

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Volume | closure color | `0`（空闭包） | 体积着色闭包，包含吸收、散射等所有体积着色信息 |

## 输出参数

无显式输出参数。该着色器直接写入开放着色语言（OSL）的内建输出变量 `Ci`。

## 实现逻辑

将输入的体积闭包直接赋值给开放着色语言（OSL）全局输出变量 `Ci`：

```osl
Ci = Volume;
```

`Ci` 是开放着色语言（OSL）中预定义的输出变量。对于体积着色器，`Ci` 承载的是体积散射和吸收闭包，Cycles 渲染器在体积光线步进（Volume Ray Marching）过程中读取这些信息。

## 对应 SVM 节点

对应 Cycles SVM（着色器虚拟机）中体积材质节点树的输出端。在 SVM 中不存在对应的独立节点。
