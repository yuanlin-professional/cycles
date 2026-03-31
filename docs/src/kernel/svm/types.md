# types.h - SVM 类型定义与枚举常量

## 概述
`types.h` 定义了着色器虚拟机(SVM)的所有核心类型，包括栈常量、节点类型枚举（通过 X-Macro 模板生成）、各类节点参数枚举以及闭包(Closure)类型系统。该文件是 SVM 系统的类型基础，几乎所有 SVM 相关文件都直接或间接依赖它。

文件还定义了一系列 `CLOSURE_IS_*` 宏，用于在运行时高效判断闭包类型的分类归属（漫反射、光泽、透射、体积等）。

## 核心函数

本文件不包含函数实现，主要定义常量、枚举和宏。

### 栈常量
- `SVM_STACK_SIZE = 255` - SVM 栈的固定大小
- `SVM_STACK_INVALID = 255` - 表示栈偏移量无效（即该参数未连接）
- `SVM_BUMP_EVAL_STATE_SIZE = 10` - 凹凸求值状态的栈空间大小

### 关键枚举类型

| 枚举 | 用途 |
|------|------|
| `ShaderNodeType` | 所有着色器节点类型（通过 `node_types_template.h` 生成） |
| `ShaderType` | 着色器类型：表面、体积、位移、凹凸 |
| `ClosureType` | 所有闭包类型：BSDF、BSSRDF、体积、Holdout 等 |
| `NodeNoiseType` | 噪声类型：多重分形、fBm、混合多重分形、脊状多重分形、异构地形 |
| `NodeGaborType` | Gabor 噪声类型：2D、3D |
| `NodeVoronoiFeature` | Voronoi 特征：F1、F2、平滑 F1、到边缘距离、N 球体半径 |
| `NodeVoronoiDistanceMetric` | Voronoi 距离度量：欧几里得、曼哈顿、切比雪夫、闵可夫斯基 |
| `NodeGradientType` | 渐变类型：线性、二次、缓动、对角线、径向、球形等 |
| `NodeWaveType` | 波纹类型：条带、环状 |
| `NodeSkyType` | 天空模型：Preetham、Hosek、单次散射、多次散射(Nishita) |
| `NodeMathType` | 数学运算类型（40+ 种运算） |
| `NodeVectorMathType` | 向量数学运算类型（27 种运算） |
| `NodeMix` | 混合模式（20 种混合方式） |

### 闭包类型分类宏
- `CLOSURE_IS_BSDF(type)` - 是否为 BSDF
- `CLOSURE_IS_BSDF_DIFFUSE(type)` - 是否为漫反射 BSDF
- `CLOSURE_IS_BSDF_GLOSSY(type)` - 是否为光泽 BSDF
- `CLOSURE_IS_BSDF_TRANSMISSION(type)` - 是否为透射 BSDF
- `CLOSURE_IS_BSSRDF(type)` - 是否为 BSSRDF（次表面散射）
- `CLOSURE_IS_VOLUME(type)` - 是否为体积闭包
- `CLOSURE_IS_HOLDOUT(type)` - 是否为 Holdout 闭包

### 阈值常量
- `CLOSURE_WEIGHT_CUTOFF = 1e-5f` - 闭包权重截断阈值
- `BSDF_ROUGHNESS_SQ_THRESH = 2e-10f` - 粗糙度平方阈值（低于此值视为奇异 BSDF）
- `THINFILM_THICKNESS_CUTOFF = 0.1f` - 薄膜厚度截断值

## 依赖关系
- **内部头文件**:
  - `kernel/svm/node_types_template.h` - 通过 `#include` 和 X-Macro 模式生成 `ShaderNodeType` 枚举
- **被引用**: `kernel/svm/svm.h`, `kernel/svm/util.h`, `kernel/svm/sky.h` 及几乎所有 SVM 相关文件

## 实现细节

### X-Macro 节点类型生成
`ShaderNodeType` 枚举通过一种称为 "X-Macro" 的模式生成：
```cpp
enum ShaderNodeType {
#define SHADER_NODE_TYPE(name) name,
#include "node_types_template.h"
  NODE_NUM
};
```
这确保了枚举定义顺序与 `svm.h` 中的 switch-case 顺序一致，对 OpenCL 性能至关重要。

### 闭包类型编码约束
`static_assert(NBUILTIN_CLOSURES < 256, ...)` 确保闭包类型总数不超过 256，因为 SVM 指令打包时使用单字节存储闭包类型。

### 闭包分类宏的设计
闭包分类宏利用枚举值的连续排列特性，通过范围比较实现 O(1) 的类型判断，注释中提到这是一种"偷懒但高效"的内存使用方式。

## 关联文件
- `kernel/svm/node_types_template.h` - 节点类型模板定义
- `kernel/svm/svm.h` - 主解释器（使用此处定义的枚举进行分发）
- `kernel/svm/util.h` - 栈操作工具（使用此处定义的 `SVM_STACK_*` 常量）
- `kernel/svm/closure.h` - 闭包节点实现（使用此处定义的 `ClosureType`）
