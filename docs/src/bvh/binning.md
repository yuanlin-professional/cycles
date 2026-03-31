# binning.h / binning.cpp - 基于分箱的表面积启发式(SAH)对象划分器

## 概述

本文件实现了单线程的对象分箱器(Object Binner),用于在层次包围体(BVH)构建过程中寻找最优分割平面。其核心思想是在每个维度上将图元质心空间均匀划分为多个箱(bin),通过统计各箱中图元的包围盒和数量,利用表面积启发式(SAH)快速评估所有候选分割位置的代价,从而选出最佳分割维度和分割位置。该算法改编自 Intel 的参考实现。

## 类与结构体

### BVHObjectBinning
- **继承**: `BVHRange`(定义于 `bvh/params.h`）
- **功能**: 对给定范围内的图元引用进行分箱,计算表面积启发式(SAH)代价,找到最优分割维度和位置,并执行图元分区操作。
- **关键成员**:
  - `splitSAH` — 最优分割的 SAH 代价
  - `leafSAH` — 创建叶节点的 SAH 代价
  - `dim` — 最优分割维度（0=X, 1=Y, 2=Z）
  - `pos` — 最优分割位置（箱索引）
  - `num_bins` — 实际使用的箱数量（最多 `MAX_BINS=32`）
  - `scale` — 将质心坐标映射到箱编号的缩放因子
  - `bounds_` — 有效包围盒（可能是非轴对齐空间下的包围盒）
  - `cent_bounds_` — 质心包围盒
  - `unaligned_heuristic_` — 非轴对齐启发式指针（用于曲线等几何体）
  - `aligned_space_` — 对齐变换空间指针
- **关键方法**:
  - `BVHObjectBinning(const BVHRange &job, BVHReference *prims, ...)` — 构造时即完成分箱和最优分割搜索
  - `split(BVHReference *prims, BVHObjectBinning &left_o, BVHObjectBinning &right_o)` — 根据最优分割位置将图元分为左右两组
  - `get_bin(const BoundBox &box)` — 计算包围盒对应的箱编号（各维度）
  - `get_bin(const float3 &c)` — 计算质心点对应的箱编号
  - `blocks(const int4 &a)` — 计算占用的块数（用于 SAH 代价计算）
  - `get_prim_bounds(const BVHReference &prim)` — 获取图元包围盒（支持非轴对齐空间）
  - `unaligned_bounds()` — 返回非轴对齐包围盒

## 核心函数

### 构造函数 `BVHObjectBinning::BVHObjectBinning`
1. 根据是否有 `aligned_space` 计算有效包围盒和质心包围盒
2. 根据图元数量动态确定箱数(`min(MAX_BINS, 4 + 0.05 * size)`)
3. 遍历所有图元,将其映射到对应的箱中,统计各箱各维度的包围盒和图元数
4. 从右向左扫描,计算右侧前缀合并包围盒面积
5. 从左向右扫描,与右侧数据结合计算每个候选分割位置的 SAH 代价
6. 选择三个维度中 SAH 最小的作为最优分割

### `BVHObjectBinning::split`
1. 使用双指针从两端遍历图元数组,根据质心所在箱编号判断归属左侧或右侧
2. 如果分割未产生进展(所有图元质心相同),退化为中位数分割(median split)

### 辅助函数（文件作用域）
- `get_best_dimension(const float4 &bestSAH)` — 从三个维度的 SAH 中选出最小值对应的维度
- `prefetch_L1/L2/L3/NTA` — 预取指令的空实现（SSE 替代方案）
- `extract<src>` / `insert<dst>` — int4/float4 元素访问的模板工具函数

## 依赖关系

- **内部头文件**:
  - `bvh/params.h` — `BVHRange`、`BVHReference`、`BVHSpatialStorage` 等基础类型
  - `bvh/unaligned.h` — `BVHUnaligned` 非轴对齐启发式
  - `util/types.h` — `int4`、`float4`、`float3` 等基本类型
  - `util/algorithm.h` — 排序和交换工具
  - `util/boundbox.h` — `BoundBox` 包围盒类
- **被引用**:
  - `bvh/build.cpp` — BVH 构建器使用分箱器进行快速分割
  - `bvh/unaligned.cpp` — 非轴对齐分割中使用分箱器

## 实现细节 / 关键算法

### 分箱 SAH 算法
传统 SAH 需要对每个图元排序后逐一评估分割代价,时间复杂度为 O(N log N)。分箱 SAH 将空间离散化为固定数量的箱(最多 32 个),将图元按质心位置映射到箱中,然后通过一次从右到左和一次从左到右的线性扫描即可完成所有候选位置的 SAH 评估,复杂度为 O(N + B),其中 B 为箱数量。

### 块数计算
SAH 代价中图元数量使用 "块数" (blocks) 计算而非直接计数,块大小为 `1 << LOG_BLOCK_SIZE = 4`,即每 4 个图元为一块。这对渲染遍历的缓存行为做了更准确的建模。

### 非轴对齐支持
当提供 `aligned_space` 时,包围盒计算会通过 `BVHUnaligned` 在变换空间中进行,这对于曲线(Hair)等非轴对齐几何体能获得更紧凑的包围盒。

### 退化处理
当所有图元质心重合导致分割无法产生进展时,`split()` 方法退化为简单的中位数分割,确保递归构建不会陷入死循环。

## 关联文件

- `bvh/params.h` — 定义 `BVHRange`、`BVHReference`、`BVHParams` 等基础类型
- `bvh/unaligned.h` / `bvh/unaligned.cpp` — 非轴对齐包围盒计算
- `bvh/build.h` / `bvh/build.cpp` — BVH 构建主流程,调用分箱器
- `bvh/split.h` / `bvh/split.cpp` — 空间分割(Spatial Split)的另一种分割策略
