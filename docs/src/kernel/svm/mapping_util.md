# mapping_util.h - 映射变换核心算法工具

## 概述

本文件提供了映射节点的核心数学变换函数 `svm_mapping`，实现了四种不同类型的向量映射变换：点映射、纹理映射、向量映射和法线映射。该文件被 `mapping.h` 引用以实现 SVM 映射节点，同时也被场景层的 `shader_nodes.cpp` 引用用于着色器节点的编译优化。

## 核心函数

### `svm_mapping`

```c
ccl_device float3 svm_mapping(
    NodeMappingType type,
    const float3 vector,
    const float3 location,
    const float3 rotation,
    const float3 scale)
```

**功能**：根据映射类型对向量执行相应的空间变换。

**参数**：
- `type`：映射类型枚举
- `vector`：输入向量
- `location`：平移量
- `rotation`：欧拉旋转角（弧度）
- `scale`：缩放因子

**四种映射类型**：

| 类型 | 枚举值 | 公式 | 说明 |
|------|--------|------|------|
| 点 | `NODE_MAPPING_TYPE_POINT` | `R * (v * s) + t` | 先缩放、再旋转、最后平移 |
| 纹理 | `NODE_MAPPING_TYPE_TEXTURE` | `R^T * (v - t) / s` | 点映射的逆运算 |
| 向量 | `NODE_MAPPING_TYPE_VECTOR` | `R * (v * s)` | 仅旋转和缩放，无平移 |
| 法线 | `NODE_MAPPING_TYPE_NORMAL` | `normalize(R * (v / s))` | 缩放取倒数，结果归一化 |

其中 `R` 为旋转矩阵，`R^T` 为其转置，`v` 为输入向量，`s` 为缩放，`t` 为平移。

## 依赖关系

- **内部头文件**：`kernel/svm/types.h`、`util/math.h`、`util/transform.h`、`util/types.h`
- **被引用**：`kernel/svm/mapping.h`、`scene/shader_nodes.cpp`

## 实现细节 / 关键算法

- **欧拉角到旋转矩阵**：使用 `euler_to_transform` 将欧拉旋转角转换为 `Transform` 矩阵。
- **纹理映射是点映射的逆操作**：纹理映射先减去平移量，然后应用旋转矩阵的转置（等价于逆旋转），最后除以缩放因子。这确保了纹理在对象上的正确投影。
- **法线映射的特殊处理**：法线在非均匀缩放下需要使用逆转置矩阵变换。这里的实现通过"除以缩放"再旋转来等效实现，最终归一化确保法线为单位向量。
- **安全除法**：使用 `safe_divide` 和 `safe_normalize` 避免除零错误。

## 关联文件

- `kernel/svm/mapping.h`：映射节点的 SVM 接口层
- `kernel/svm/types.h`：定义 `NodeMappingType` 枚举
- `util/transform.h`：提供 `Transform` 结构和 `euler_to_transform` 等变换函数
- `scene/shader_nodes.cpp`：场景层着色器节点编译时直接调用 `svm_mapping`
