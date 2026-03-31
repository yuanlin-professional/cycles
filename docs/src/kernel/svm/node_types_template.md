# node_types_template.h - SVM 节点类型 X-Macro 模板

## 概述
`node_types_template.h` 使用 X-Macro 模式定义了所有着色器虚拟机(SVM)节点类型。该文件不是传统的头文件，而是一个宏展开模板——在被包含前，调用方需先定义 `SHADER_NODE_TYPE(name)` 宏，然后包含此文件，模板会为每个节点类型展开一次该宏。文件末尾自动 `#undef SHADER_NODE_TYPE`。

此模板的核心作用是确保节点类型枚举的定义顺序与 `svm.h` 中 switch-case 的顺序严格一致，这对 OpenCL 性能优化至关重要。

## 核心函数

本文件无函数定义，仅包含宏展开列表。

### 节点类型列表（共约 112 个）

**控制流节点**:
- `NODE_END` - 着色器程序结束
- `NODE_SHADER_JUMP` - 按着色器类型（表面/体积/位移）跳转
- `NODE_JUMP_IF_ZERO` / `NODE_JUMP_IF_ONE` - 条件跳转

**闭包节点**:
- `NODE_CLOSURE_BSDF` - BSDF 闭包
- `NODE_CLOSURE_EMISSION` - 发射闭包
- `NODE_CLOSURE_BACKGROUND` - 背景闭包
- `NODE_CLOSURE_VOLUME` - 体积闭包
- `NODE_CLOSURE_HOLDOUT` - Holdout 闭包
- `NODE_MIX_CLOSURE` - 混合闭包
- `NODE_CLOSURE_SET_WEIGHT` / `NODE_CLOSURE_WEIGHT` - 闭包权重设置

**纹理节点**:
- `NODE_TEX_IMAGE` / `NODE_TEX_IMAGE_BOX` - 图像纹理
- `NODE_TEX_NOISE` - 噪声纹理
- `NODE_TEX_VORONOI` - Voronoi 纹理
- `NODE_TEX_GABOR` - Gabor 纹理
- `NODE_TEX_WAVE` - 波纹纹理
- `NODE_TEX_MAGIC` - 魔法纹理
- `NODE_TEX_CHECKER` - 棋盘格纹理
- `NODE_TEX_BRICK` - 砖块纹理
- `NODE_TEX_GRADIENT` - 渐变纹理
- `NODE_TEX_SKY` - 天空纹理
- `NODE_TEX_ENVIRONMENT` - 环境纹理
- `NODE_TEX_WHITE_NOISE` - 白噪声纹理

**几何与属性节点**:
- `NODE_GEOMETRY` / `NODE_GEOMETRY_BUMP_DX` / `NODE_GEOMETRY_BUMP_DY`
- `NODE_ATTR` / `NODE_ATTR_BUMP_DX` / `NODE_ATTR_BUMP_DY`
- `NODE_VERTEX_COLOR` / `NODE_VERTEX_COLOR_BUMP_DX` / `NODE_VERTEX_COLOR_BUMP_DY`
- `NODE_TEX_COORD` / `NODE_TEX_COORD_BUMP_DX` / `NODE_TEX_COORD_BUMP_DY`

**运算节点**:
- `NODE_MATH` / `NODE_VECTOR_MATH`
- `NODE_MIX` / `NODE_MIX_COLOR` / `NODE_MIX_FLOAT` / `NODE_MIX_VECTOR`
- `NODE_MAP_RANGE` / `NODE_VECTOR_MAP_RANGE` / `NODE_CLAMP`

**填充节点**:
- `NODE_PAD1` / `NODE_PAD2` - 结构体对齐填充

## 依赖关系
- **内部头文件**: 无（自包含模板）
- **被引用**: `kernel/svm/types.h`（生成 `ShaderNodeType` 枚举）

## 实现细节

### X-Macro 工作原理
```cpp
// 在 types.h 中的使用方式：
enum ShaderNodeType {
#define SHADER_NODE_TYPE(name) name,
#include "node_types_template.h"
  NODE_NUM  // 枚举计数
};
```
每次 `#include` 时，`SHADER_NODE_TYPE` 宏会被展开为不同的代码（如枚举成员、case 标签等），这种模式允许从单一来源生成多种代码结构。

### 顺序约束
文件头部注释明确指出：枚举定义顺序必须与 `svm.h` 中 switch-case 的顺序匹配。这是因为 OpenCL 编译器对 switch 分支的优化依赖于 case 值的连续性和有序性。

### 自动清理
文件末尾的 `#undef SHADER_NODE_TYPE` 确保宏不会泄漏到后续代码中，符合 X-Macro 的最佳实践。

## 关联文件
- `kernel/svm/types.h` - 使用此模板生成 `ShaderNodeType` 枚举
- `kernel/svm/svm.h` - switch-case 顺序必须与此模板一致
