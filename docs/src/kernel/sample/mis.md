# mis.h - 多重重要性采样(MIS)权重计算

## 概述
本文件实现了多重重要性采样（Multiple Importance Sampling, MIS）的权重启发式函数。MIS 是路径追踪中组合多种采样策略的核心技术，通过为每种策略赋予合理权重，有效降低方差并避免"萤火虫"噪点。Cycles 提供了三种经典的 MIS 启发式：平衡启发式、功率启发式和最大值启发式，其中功率启发式是最常用的默认选择。代码源自 Open Shading Language。

## 类与结构体
无。

## 枚举与常量
无。

## 核心函数

### balance_heuristic()
- **签名**: `ccl_device float balance_heuristic(const float a, const float b)`
- **功能**: 计算二路平衡启发式权重。公式为 `a / (a + b)`，其中 `a` 和 `b` 通常为各采样策略的 `n_i * p_i`（样本数乘以对应 PDF）。平衡启发式是 MIS 理论中证明最优的无偏组合方式。

### balance_heuristic_3()
- **签名**: `ccl_device float balance_heuristic_3(const float a, const float b, float c)`
- **功能**: 计算三路平衡启发式权重。公式为 `a / (a + b + c)`。用于同时组合三种采样策略的场景（如直接光照中的 BSDF 采样 + 光源采样 + 体积采样）。

### power_heuristic()
- **签名**: `ccl_device float power_heuristic(const float a, const float b)`
- **功能**: 计算二路功率启发式权重（指数为 2）。公式为 `a^2 / (a^2 + b^2)`。功率启发式相比平衡启发式更激进地偏向高 PDF 的策略，在实践中通常能获得更低的方差，是 Cycles 中最常用的 MIS 权重计算方式。

### power_heuristic_3()
- **签名**: `ccl_device float power_heuristic_3(const float a, const float b, float c)`
- **功能**: 计算三路功率启发式权重。公式为 `a^2 / (a^2 + b^2 + c^2)`。

### max_heuristic()
- **签名**: `ccl_device float max_heuristic(const float a, const float b)`
- **功能**: 最大值启发式，采用"赢者通吃"策略：若 `a > b` 返回 1，否则返回 0。这是最极端的权重分配方式，仅将全部权重给予 PDF 较大的策略。在特殊场景下可作为调试或对比用途。

## 依赖关系
- **内部头文件**: `util/defines.h`
- **被引用**:
  - `src/kernel/light/sample.h` - 光源采样中使用 MIS 权重组合直接光照的 BSDF 和光源 PDF

## 实现细节 / 关键算法

### MIS 理论背景
多重重要性采样由 Veach (1995) 提出，核心思想是：当同时有多种采样策略可用时（如路径追踪中的光源采样和 BSDF 采样），与其选择单一策略，不如将所有策略的样本加权组合。MIS 权重函数满足归一化约束，确保最终估计器无偏。

### 启发式对比
- **平衡启发式** `w = p_i / sum(p_j)`：理论最优，但实践中对低 PDF 的策略给予了过多权重
- **功率启发式** `w = p_i^beta / sum(p_j^beta)`（`beta=2`）：对接近零的 PDF 衰减更快，减少了那些采样效率低的策略对方差的贡献
- **最大值启发式**：功率启发式在 `beta -> inf` 时的极限形式

### 参数含义
所有函数的参数 `a`、`b`（、`c`）代表的是各采样策略的 `n_i * pdf_i` 值，即该策略的采样数量乘以在当前样本点处的概率密度。在 Cycles 中，通常每种策略只取一个样本（`n_i = 1`），因此参数直接就是 PDF 值。

## 关联文件
- `src/kernel/light/sample.h` - 光源采样的主要调用方，使用 MIS 组合光源 PDF 与 BSDF PDF
- `src/kernel/sample/mapping.h` - 提供各种 PDF 计算函数（如 `pdf_cos_hemisphere`）
- `src/kernel/integrator/shade_surface.h` - 表面着色积分器，间接通过光源采样使用 MIS
