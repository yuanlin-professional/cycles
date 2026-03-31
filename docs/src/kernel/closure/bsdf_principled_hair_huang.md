# bsdf_principled_hair_huang.h - Huang 微面元毛发散射模型实现

## 概述

本文件实现了基于 Huang, Hullin & Hanika (2022) 论文 "A Microfacet-based Hair Scattering Model" 的毛发双向散射分布函数（BSDF）。与传统的 Chiang 模型不同，该模型使用微面元（Microfacet）理论来描述毛发表面的散射行为，支持椭圆形截面和近场/远场自适应渲染。各散射分量（R、TT、TRT、TRRT+）通过在毛发截面上的数值积分（复合 Simpson 法则）进行求值。这是 Cycles 中 Principled Hair BSDF 的新一代实现。

## 类与结构体

### `struct HuangHairExtra`
扩展数据结构，存储运行时计算的辅助信息：
- `R`, `TT`, `TRT` — 各散射分量的可选调制因子
- `Y`, `Z` — 局部坐标系轴（X 轴存储在 `bsdf->N` 中）
- `wi` — 局部坐标系下的入射方向
- `radius` — 从视线方向投影的毛发半径
- `e2` — 离心率的平方，`1 - aspect_ratio^2`
- `pixel_coverage` — 在 `h` 空间中半个像素的投影宽度
- `h` — 有效积分区间（已预除以半径，范围 [-1, 1]）

### `struct HuangHairBSDF`
Huang 毛发 BSDF 数据结构，继承自 `SHADER_CLOSURE_BASE`：
- `sigma` — 吸收系数（光谱值），控制毛发颜色
- `roughness` — 微面元分布粗糙度
- `tilt` — 角质层倾斜角度
- `eta` — 折射率
- `aspect_ratio` — 短轴与长轴之比（1.0 为圆形截面）
- `h` — 方位角偏移量
- `extra` — 指向 `HuangHairExtra` 的指针

## 核心函数

### 坐标系工具函数
- **`sin_theta(w)`, `cos_theta(w)`, `tan_theta(w)`** — 从方向向量提取极角三角函数值。
- **`sin_phi(w)`, `sincos_phi(w)`** — 从方向向量提取方位角三角函数值。
- **`dir_theta(w)`, `dir_phi(w)`, `dir_sph(w)`** — 从方向向量提取球坐标角度。
- **`is_circular(b)`** — 判断截面是否为圆形（`b == 1.0`）。

### 椭圆截面几何
- **`to_phi(gamma, b)`, `to_gamma(phi, b)`** — 椭圆截面上 `gamma` 角与 `phi` 角之间的转换。
- **`phi_to_h(phi, b, wi)`** — 将方位角转换为 `h` 值（入射位置参数）。
- **`h_to_gamma(h_div_r, b, wi)`** — 将 `h` 值转换为 `gamma` 角。
- **`d_gamma_d_h(sincos_phi_i, gamma, b)`** — 雅可比行列式 `|d_gamma/d_h|`，用于积分变量替换。
- **`to_point(gamma, b)`** — 计算椭圆上给定 `gamma` 角的坐标点。
- **`sphg_dir(theta, gamma, b)`** — 由 `theta` 和 `gamma` 构建方向向量。
- **`arc_length(e2, gamma)`** — 椭圆弧长因子。

### 微面元采样与可见性
- **`sample_wh(roughness, wi, wm, rand)`** — 从倾斜的介观法线（mesonormal）`wm` 出发，采样 GGX 微面元法线。
- **`microfacet_visible(v, m, h)` / `microfacet_visible(wi, wo, m, h)`** — 检查微法线/介观法线从给定方向的直接可见性。
- **`bsdf_Go(alpha2, cos_NI, cos_NO)`** — 联合遮蔽-阴影项除以入射方向遮蔽项。

### 能量校正
- **`bsdf_hair_huang_energy_scale(kg, mu, rough, ior)`** — 基于玻璃材质查找表的反照率校正因子。
- **`is_nearfield(bsdf)`** — 判断是否使用近场模型（投影半径 > 像素覆盖范围时）。

### 散射求值
- **`bsdf_hair_huang_eval_r(kg, sc, wi, wo)`** — R（主反射）分量求值。通过复合 Simpson 1/3 法则在 `h` 区间上积分，积分核包含 GGX 微面元分布 D、遮蔽-阴影项 G、弧长因子和能量校正因子。
- **`bsdf_hair_huang_eval_trrt(T, R, A)`** — TRRT+ 残余分量近似，通过几何级数求和。
- **`bsdf_hair_huang_eval_residual(kg, sc, wi, wo, rng_quadrature)`** — TT 和 TRT 分量求值。使用混合蒙特卡洛-Simpson 积分法：Simpson 法则用于 `h` 维度积分，蒙特卡洛用于微法线采样。TRRT+ 项也在此函数中累计。

### 闭包接口
- **`bsdf_hair_huang_setup(sd, bsdf, path_flag)`** — 闭包初始化：计算局部坐标系、投影半径、离心率；处理椭圆截面的主次轴；当交点在投影半径外时退化为透明材质。
- **`bsdf_hair_huang_eval(kg, sd, sc, wo, pdf)`** — 完整 BSDF 求值：计算可见方位角范围、处理近场/远场积分区间、合并 R 和残余分量。PDF 设为 1.0（重要性采样已内化在值中）。
- **`bsdf_hair_huang_sample(kg, sc, sd, rand, eval, wo, pdf, sampled_roughness)`** — BSDF 采样：依次采样 R、TT、TRT、TRRT+ 各分量的出射方向和能量，按能量比选择叶瓣。
- **`bsdf_hair_huang_blur(sc, roughness)`** — 实现 Filter Glossy。
- **`bsdf_hair_huang_albedo(sd, sc)`** — 毛发反照率近似，假设圆形截面和镜面反射，通过几何级数求和。

## 依赖关系
- **内部头文件**:
  - `kernel/types.h` — 内核类型定义
  - `kernel/closure/bsdf_microfacet.h` — GGX 分布函数、遮蔽-阴影项和 VNDF 采样
  - `kernel/closure/bsdf_principled_hair_chiang.h` — 复用 `longitudinal_scattering` 纵向散射函数
  - `kernel/closure/bsdf_transparent.h` — 透明闭包（退化情况使用）
  - `kernel/closure/bsdf_util.h` — BSDF 工具函数
  - `kernel/sample/lcg.h` — 线性同余随机数生成器
- **被引用**:
  - `kernel/closure/bsdf.h` — 闭包统一调度入口

## 实现细节 / 关键算法

### 条件编译
`bsdf_hair_huang_setup` 被 `#ifdef __HAIR__` 包裹，仅在启用毛发渲染功能时编译。

### 微面元毛发散射模型
与 Chiang 模型使用解析散射函数不同，Huang 模型将毛发表面视为微面元表面，各散射分量通过以下流程计算：
1. 在毛发截面上选取入射点（`h`），确定介观法线（mesonormal）
2. 从介观法线采样微法线（micronormal）
3. 根据微法线计算反射或折射方向
4. 通过 Beer 定律计算路径吸收
5. 在截面上积分所有入射点的贡献

### 数值积分策略
- **R 分量**: 使用复合 Simpson 1/3 法则在 `h` 区间上确定性积分。积分分辨率由粗糙度决定（`res = roughness * 0.7`），区间数保持为偶数。
- **TT/TRT 分量**: 使用混合方法——外层（`h` 维度）为 Simpson 积分，内层（微法线采样）为蒙特卡洛采样，随机数来自 LCG 随机数生成器。
- **TRRT+ 分量**: 采用 Chiang 模型的 `longitudinal_scattering` 函数近似，方位角假设为均匀分布。

### 椭圆截面支持
通过 `aspect_ratio` 参数支持非圆形毛发截面。关键适配包括：
- `gamma` 到 `phi` 的非线性映射：`phi = atan2(b * sin_gamma, cos_gamma)`
- 弧长因子：`arc_length = sqrt(1 - e2 * sin^2(gamma))`
- 吸收路径长度根据椭圆几何计算

### 近场/远场自适应
受 Yan et al. "An Efficient and Practical Near and Far Field Fur Reflectance Model" 启发：
- **远场模式**: 当投影半径小于像素覆盖范围时，在整个可见 `h` 范围上均匀采样
- **近场模式**: 将积分区间缩小到当前像素可见的子区间，获得更高的细节精度

### 透明退化
当交点位于投影半径之外（`|h| >= radius`）时，闭包退化为完全透明（`bsdf_transparent_setup`），释放已分配的闭包槽位。

### 采样策略
采样过程依次追踪光线在毛发内的路径：
1. 采样 `h`（远场时均匀随机，近场时使用交点）
2. 采样 `wh1`（入射微法线）得到 R 方向和折射方向 `wt`
3. 采样 `wh2`（内部微法线）得到 TT 方向和内部反射方向 `wtr`
4. 采样 `wh3`（再次出射微法线）得到 TRT 方向
5. TRRT+ 使用纵向散射函数采样
6. 按各分量能量权重选择最终出射方向

## 关联文件
- `kernel/closure/bsdf_microfacet.h` — 提供 GGX 分布 `bsdf_D`、遮蔽项 `bsdf_G`、`bsdf_lambda`、VNDF 采样 `microfacet_ggx_sample_vndf`
- `kernel/closure/bsdf_principled_hair_chiang.h` — 提供 `longitudinal_scattering` 用于 TRRT+ 近似
- `kernel/closure/bsdf_transparent.h` — 透明闭包，用于截面外退化
- `kernel/closure/bsdf.h` — 闭包调度入口
- `kernel/sample/lcg.h` — LCG 随机数生成器，用于积分中的蒙特卡洛采样
