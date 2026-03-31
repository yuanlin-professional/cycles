# node_output_surface.osl - 表面输出着色器

## 概述

表面输出着色器（Surface Output Node）是开放着色语言（OSL）中的输出类节点，用于将计算好的表面闭包（Closure）赋值给渲染器的最终表面输出。它是材质节点树的终端节点，将所有表面着色计算结果传递给 Cycles 渲染器。

## 着色器签名

```osl
surface node_output_surface(
    closure color Surface = 0
)
```

注意：该着色器类型为 `surface`，而非通用 `shader`。

## 输入参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| Surface | closure color | `0`（空闭包） | 表面着色闭包，包含 BSDF、发光等所有表面着色信息 |

## 输出参数

无显式输出参数。该着色器直接写入开放着色语言（OSL）的内建输出变量 `Ci`。

## 实现逻辑

将输入的表面闭包直接赋值给开放着色语言（OSL）全局输出变量 `Ci`：

```osl
Ci = Surface;
```

`Ci` 是开放着色语言（OSL）中预定义的表面着色器输出变量，表示最终的颜色闭包积分（Color Integral）。Cycles 渲染器从 `Ci` 中读取完整的表面着色信息进行光线追踪计算。

## 对应 SVM 节点

对应 Cycles SVM（着色器虚拟机）中材质节点树的输出端。在 SVM 中不存在对应的独立节点，因为 SVM 的着色器栈本身就是直接输出最终结果。
