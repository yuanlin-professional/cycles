# bsdf_sheen.h - 光泽（Sheen）双向散射分布函数(BSDF)闭包

## 概述

本文件实现了 Cycles 渲染器中的光泽（Sheen）BSDF 闭包，基于 Tizian Zeltner、Brent Burley 和 Matt Jen-Yuan Chiang 于 2022 年发表的论文"Practical Multiple-Scattering Sheen Using Linearly Transformed Cosines"。该模型通过线性变换余弦（LTC）方法高效近似多重散射光泽效果，常用于模拟布料、织物等材质边缘处的柔和反射光。

## 类与结构体

### `SheenBsdf`
光泽 BSDF 闭包的数据结构，继承自 `SHADER_CLOSURE_BASE`。

| 成员 | 类型 | 说明 |
|------|------|------|
| `roughness` | `float` | 粗糙度参数，控制光泽扩散范围 |
| `transformA` | `float` | LTC 变换矩阵参数 A |
| `transformB` | `float` | LTC 变换矩阵参数 B |
| `T` | `float3` | 切线向量 |
| `B` | `float3` | 副切线向量 |

编译时通过 `static_assert` 确保 `SheenBsdf` 不超过 `ShaderClosure` 的大小。

## 核心函数

### `bsdf_sheen_setup`
```cpp
ccl_device int bsdf_sheen_setup(KernelGlobals kg,
                                const ccl_private ShaderData *sd,
                                ccl_private SheenBsdf *bsdf)
```
初始化光泽 BSDF。主要流程：
1. 设置闭包类型为 `CLOSURE_BSDF_SHEEN_ID`
2. 将粗糙度钳制到 `[1e-3, 1.0]` 范围
3. 基于法线和入射方向构建正交切线空间 `(T, B)`
4. 计算入射角余弦 `cosNI`
5. 从预计算的 LTC 查找表中读取变换参数 `transformA`、`transformB` 和 `albedo`
6. 若变换参数或反照率过小（无效 LTC），则跳过该闭包
7. 将反照率 `albedo` 乘入权重，确保能量守恒

### `bsdf_sheen_eval`
```cpp
ccl_device Spectrum bsdf_sheen_eval(const ccl_private ShaderClosure *sc,
                                    const float3 /*wi*/,
                                    const float3 wo,
                                    ccl_private float *pdf)
```
对给定出射方向求值。将出射方向转换到局部坐标系后，通过 LTC 变换计算概率密度值。

### `bsdf_sheen_sample`
```cpp
ccl_device int bsdf_sheen_sample(const ccl_private ShaderClosure *sc,
                                 const float3 Ng,
                                 const float3 /*wi*/,
                                 const float2 rand,
                                 ccl_private Spectrum *eval,
                                 ccl_private float3 *wo,
                                 ccl_private float *pdf)
```
重要性采样。在均匀圆盘上采样后通过 LTC 逆变换得到出射方向。返回标签 `LABEL_REFLECT | LABEL_DIFFUSE`。

## 依赖关系
- **内部头文件**:
  - `kernel/sample/mapping.h` — 采样映射工具（`sample_uniform_disk` 等）
  - `kernel/util/lookup_table.h` — 查找表读取工具（`lookup_table_read_2D`）
- **被引用**:
  - `src/kernel/closure/bsdf.h` — BSDF 调度总入口

## 实现细节 / 关键算法

### LTC（线性变换余弦）方法
核心思想是将标准余弦分布通过线性变换来近似复杂的 BRDF 分布。变换由参数 `(a, b)` 控制：
- 求值时：将出射方向变换到 LTC 空间，计算变换后的余弦分布值
- 采样时：先在均匀圆盘上采样，再通过逆变换映射到实际出射方向

关键公式（局部坐标系下）：
```
lenSqr = (a * localO.x + b * localO.z)^2 + (a * localO.y)^2 + localO.z^2
pdf = (1/pi) * max(localO.z, 0) * (a / lenSqr)^2
```

### 预计算查找表
LTC 参数通过 32x32 的二维查找表存储，索引为 `(cosNI, roughness)`。表中包含三组数据：
- `transformA`：LTC 变换参数 A
- `transformB`：LTC 变换参数 B
- `albedo`：反照率（用于能量守恒修正）

## 关联文件
- `src/kernel/closure/bsdf.h` — BSDF 统一调度
- `src/kernel/util/lookup_table.h` — 查找表基础设施
- `src/kernel/sample/mapping.h` — 采样空间映射函数
