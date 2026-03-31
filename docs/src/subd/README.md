# subd - 细分曲面

## 概述

`subd` 模块实现了 Cycles 路径追踪渲染器中的细分曲面系统。该模块的核心职责是将细分曲面网格（Subdivision Surface Mesh）转化为可供渲染使用的三角形网格。它支持两种细分模式：

1. **线性细分（Linear）**：对四边形和多边形进行简单的线性插值细分。
2. **Catmull-Clark 细分**：通过集成 OpenSubdiv 库，实现基于 Catmull-Clark 方案的平滑细分。

模块的整体流程为：**Patch 生成 -> 自适应分裂（Split）-> 网格切分（Dice）-> 属性插值（Interpolation）**。在 Cycles 架构中，`subd` 模块被 `scene` 模块（主要是 `mesh_subdivision.cpp`）调用，将场景中标记为细分曲面的 `Mesh` 对象转化为最终用于渲染的三角形数据。

## 目录结构

| 文件名 | 类型 | 说明 |
|--------|------|------|
| `CMakeLists.txt` | 构建 | CMake 构建配置，编译为 `cycles_subd` 静态库 |
| `patch.h` / `patch.cpp` | 核心 | Patch 基类及具体实现（`LinearQuadPatch`、`BicubicPatch`），提供参数化曲面求值 |
| `subpatch.h` | 核心 | `SubPatch` 和 `SubEdge` 数据结构，表示分裂后的子面片及其边 |
| `split.h` / `split.cpp` | 核心 | `DiagSplit` 类，实现 DiagSplit 自适应分裂算法，确定边的细分因子并递归分裂面片 |
| `dice.h` / `dice.cpp` | 核心 | `EdgeDice` 类及 `SubdParams` 参数，将分裂后的子面片切分为三角形网格 |
| `interpolation.h` / `interpolation.cpp` | 核心 | `SubdAttributeInterpolation` 类，处理顶点、角点、面属性在细分过程中的插值 |
| `osd.h` / `osd.cpp` | 可选 | OpenSubdiv 集成（需 `WITH_OPENSUBDIV` 编译宏），提供 Catmull-Clark 细分的拓扑精炼和 patch 求值 |

## 核心类与数据结构

### Patch（基类）
- **定义位置**: `patch.h`
- **功能**: 参数化曲面的抽象基类，所有类型的 patch 都需要实现 `eval()` 虚方法
- **关键成员**:
  - `patch_index`: patch 索引（对应 PTex face ID）
  - `shader`: 该 patch 使用的着色器索引
  - `smooth`: 是否使用平滑法线
  - `from_ngon`: 是否来自 N 边形面的分割
- **关键方法**:
  - `eval(P, dPdu, dPdv, N, u, v)`: 纯虚方法，在参数坐标 (u, v) 处求值位置 P、偏导数 dPdu/dPdv 和法线 N

### LinearQuadPatch
- **定义位置**: `patch.h` / `patch.cpp`
- **功能**: 线性四边形 patch，使用 4 个控制顶点进行双线性插值
- **关键成员**: `hull[4]` — 四个角点的 float3 坐标
- **求值方式**: 对两组对边分别线性插值（`interp`），法线通过偏导数的叉积计算

### BicubicPatch
- **定义位置**: `patch.h` / `patch.cpp`
- **功能**: 双三次 Bezier patch，使用 16 个控制顶点
- **关键成员**: `hull[16]` — 4x4 控制网格的 float3 坐标
- **求值方式**: 使用 De Casteljau 算法（`decasteljau_bicubic`）进行双三次曲面求值

### OsdPatch
- **定义位置**: `osd.h` / `osd.cpp`（需 `WITH_OPENSUBDIV`）
- **功能**: 基于 OpenSubdiv 的 Catmull-Clark patch，继承自 `Patch`
- **关键成员**: `osd_data` — 引用 `OsdData`，包含精炼后的拓扑和 patch 表
- **求值方式**: 通过 `Far::PatchMap::FindPatch` 定位 patch，然后使用 `EvaluateBasis` 求值权重

### SubEdge
- **定义位置**: `subpatch.h`
- **功能**: 表示子面片的一条边，记录细分因子和顶点索引
- **关键成员**:
  - `start_vert_index` / `end_vert_index`: 边的起止顶点索引
  - `mid_vert_index`: 若边被分裂，中间顶点的索引
  - `T`: 边的细分段数（tessellation factor），`DSPLIT_NON_UNIFORM`（-1）表示需要进一步分裂
  - `length`: 估计的边长度，用于确定优先分裂方向
  - `depth`: 该边经过的分裂深度
- **关键常量**:
  - `DSPLIT_NON_UNIFORM = -1`: 非均匀标记，表示边需要继续分裂
  - `DSPLIT_MAX_DEPTH = 16`: 最大递归分裂深度
  - `DSPLIT_MAX_SEGMENTS = 8`: 单条边允许的最大段数

### SubPatch
- **定义位置**: `subpatch.h`
- **功能**: 表示分裂后的子面片（可以是四边形或三角形），是 `DiagSplit` 与 `EdgeDice` 之间传递数据的核心结构
- **关键成员**:
  - `patch`: 指向父 `Patch` 对象
  - `shape`: 面片形状，`TRIANGLE` 或 `QUAD`
  - `uvs[4]`: patch 参数空间内的 UV 坐标（逆时针排列）
  - `edges[4]`: 四条边，每条边包含 `SubEdge` 指针、方向标记和所有权标记
  - `inner_grid_vert_offset`: 内部网格顶点的起始索引
  - `triangles_offset`: 该子面片三角形数据的起始索引
- **关键方法**:
  - `map_uv(uv)`: 将子面片局部 UV 映射到父 patch 的参数坐标
  - `calc_num_inner_verts()`: 计算内部网格的顶点数量
  - `calc_num_triangles()`: 计算该子面片将生成的三角形数量

### SubdParams
- **定义位置**: `dice.h`
- **功能**: 细分参数的封装结构
- **关键成员**:
  - `mesh`: 目标网格
  - `ptex`: 是否生成 PTex 坐标
  - `test_steps`: 边长度估计的采样步数（默认 3）
  - `split_threshold`: 分裂阈值（默认 1）
  - `dicing_rate`: 切分率，控制三角形密度（以像素为单位）
  - `max_level`: 最大细分层级（默认 12）
  - `camera`: 摄像机指针，用于视角自适应细分
  - `objecttoworld`: 物体到世界空间的变换矩阵

### DiagSplit
- **定义位置**: `split.h` / `split.cpp`
- **功能**: 实现 DiagSplit 自适应分裂算法，将 patch 递归分裂为大小合适的子面片
- **关键成员**:
  - `subpatches`: 分裂完成的所有 `SubPatch` 列表
  - `edges`: 使用哈希集合存储的所有 `SubEdge`，保证共享边的水密性
  - `num_verts` / `num_triangles`: 分裂后的总顶点和三角形数量
- **关键方法**:
  - `split_patches(patches, stride)`: 入口方法，遍历网格所有面进行分裂
  - `split_quad(sub)`: 递归分裂四边形子面片
  - `split_triangle(sub)`: 递归分裂三角形子面片
  - `split_ngon(face, ...)`: 将 N 边形分解为 N 个四边形子面片后分裂
  - `T(patch, uv_start, uv_end, depth)`: 计算边的细分因子（tessellation factor）

### EdgeDice
- **定义位置**: `dice.h` / `dice.cpp`
- **功能**: 将分裂后的子面片切分为三角形网格，采用 DX11 风格的 EdgeDice 方式，保证相邻面片之间无缝隙（watertight）
- **关键成员**:
  - `params`: 细分参数
  - `interpolation`: 属性插值器的引用
  - `mesh_P` / `mesh_N` / `mesh_triangles` 等: 直接写入目标网格的数据指针
- **关键方法**:
  - `dice(split)`: 入口方法，使用 TBB 并行处理所有子面片的顶点坐标计算和三角形生成
  - `quad_dice(sub)` / `tri_dice(sub)`: 分别处理四边形和三角形子面片的切分
  - `add_grid_triangles_and_stitch(sub, Mu, Mv)`: 生成内部网格三角形，并将内部网格与边界缝合

### SubdAttributeInterpolation
- **定义位置**: `interpolation.h` / `interpolation.cpp`
- **功能**: 在细分过程中将顶点属性、角点属性和面属性正确插值到新生成的网格上
- **关键成员**:
  - `vertex_attributes`: 顶点属性插值回调列表（位置、法线等逐顶点属性）
  - `triangle_attributes`: 三角形属性插值回调列表（UV、颜色等逐角点/逐面属性）
- **关键方法**:
  - `setup()`: 遍历所有细分属性，为每个属性创建对应的插值回调函数
  - `setup_attribute_vertex_linear<T>()`: 线性顶点属性插值（四边形双线性插值，N 边形加权中心插值）
  - `setup_attribute_vertex_smooth<T>()`: 平滑顶点属性插值（通过 OpenSubdiv 精炼）
  - `setup_attribute_corner_linear<T>()` / `setup_attribute_corner_smooth<T>()`: 角点属性的线性/平滑插值
  - `setup_attribute_face<T>()`: 面属性直接复制到所有子三角形

### OsdMesh
- **定义位置**: `osd.h` / `osd.cpp`（需 `WITH_OPENSUBDIV`）
- **功能**: Cycles 网格到 OpenSubdiv `TopologyRefiner` 的适配器
- **关键成员**:
  - `mesh`: 引用 Cycles 的 `Mesh` 对象
  - `merged_fvars`: 面变量（Face-Varying）属性列表，对同顶点相同值的角点进行合并
- **关键方法**:
  - `sdc_options()`: 根据网格设置生成 OpenSubdiv 的细分选项（边界插值、FVar 插值模式等）
  - `use_smooth_fvar(attr)`: 判断某属性是否需要平滑 FVar 插值（通常是 UV 贴图）

### OsdData
- **定义位置**: `osd.h` / `osd.cpp`（需 `WITH_OPENSUBDIV`）
- **功能**: 存储 OpenSubdiv 精炼后的数据结构
- **关键成员**:
  - `refiner`: `Far::TopologyRefiner` — 拓扑精炼器
  - `patch_table`: `Far::PatchTable` — patch 表，存储自适应精炼后的 patch 数据
  - `patch_map`: `Far::PatchMap` — patch 查找映射，用于在给定 (face, u, v) 时快速定位 patch
  - `refined_verts`: 精炼后的所有层级顶点坐标
- **关键方法**:
  - `build(osd_mesh)`: 执行自适应精炼和 patch 表构建的完整流程

## 模块架构

整体处理流程如下：

```
输入: Mesh (含细分面、顶点、属性)
          |
          v
  [1] Patch 生成
      - 线性模式: 为每个四边形面生成 LinearQuadPatch
      - Catmull-Clark 模式: 通过 OsdData::build() 生成 OsdPatch
          |
          v
  [2] DiagSplit 自适应分裂
      - split_patches() 遍历所有面
      - 四边形面: split_quad() 递归分裂
      - N 边形面: split_ngon() 先分解为 N 个四边形再分裂
      - 每条边计算细分因子 T（基于相机投影的像素尺寸）
      - 边的 T 值过大或不均匀时触发分裂
          |
          v
  [3] EdgeDice 网格切分
      - dice() 使用 TBB 并行处理所有 SubPatch
      - 先设置边界顶点坐标（set_sides）
      - 再生成内部网格和三角形（quad_dice / tri_dice）
      - 内部网格与边界通过 stitch 算法缝合，保证水密性
          |
          v
  [4] SubdAttributeInterpolation 属性插值
      - 在 EdgeDice 的 set_vertex / set_triangle 中回调
      - 线性模式: 双线性/重心坐标插值
      - Catmull-Clark 模式: 通过 OpenSubdiv 的 patch basis 求值
          |
          v
输出: 三角形网格 (顶点、三角形索引、法线、UV 等属性)
```

DiagSplit 的自适应分裂保证了以下关键特性：
- **无裂缝（Crack-free）**: 共享边使用统一的 `SubEdge` 对象和细分因子，确保相邻面片边界顶点完全一致
- **自适应密度**: 通过相机投影估计屏幕空间像素覆盖率，近处和曲率大的区域生成更密的三角形
- **水密缝合（Watertight Stitching）**: 边界缝合时选择最短对角线方向来创建三角形，保持合理的三角形形状

## 依赖关系

### 上游依赖（本模块依赖）

| 模块/库 | 用途 |
|---------|------|
| `scene/mesh.h` | `Mesh` 类，提供细分面拓扑、顶点数据和属性 |
| `scene/camera.h` | `Camera` 类，用于视角自适应的屏幕空间尺寸计算 |
| `scene/attribute.h` | `Attribute` 类，属性数据的存储和类型信息 |
| `util/transform.h` | 变换矩阵操作 |
| `util/types.h` | 基础类型（`float2`、`float3` 等） |
| `util/boundbox.h` | `BoundBox` 包围盒 |
| `util/tbb.h` | TBB 并行化支持（`parallel_for`） |
| `util/vector.h` / `util/set.h` | 容器类型 |
| OpenSubdiv（可选） | `Far::TopologyRefiner`、`Far::PatchTable` 等，提供 Catmull-Clark 精炼和 patch 求值 |

### 下游依赖（依赖本模块）

| 模块 | 文件 | 用途 |
|------|------|------|
| `scene` | `mesh_subdivision.cpp` | 主要调用方，驱动整个细分流程（创建 Patch、调用 DiagSplit 和 EdgeDice） |
| `scene` | `mesh.h` | 包含 `dice.h`，使用 `SubdParams` 作为网格成员 |
| `scene` | `mesh.cpp` | 包含 `split.h`，在网格处理中引用分裂相关类型 |
| `scene` | `geometry.cpp` | 包含 `split.h`，在几何体处理中引用分裂相关类型 |

## 关键算法与实现细节

### DiagSplit 自适应分裂

基于论文 "DiagSplit: Parallel, Crack-free, Adaptive Tessellation for Micro-polygon Rendering"。核心思想：

1. **边细分因子计算** (`T()` 方法):
   - 沿边采样多个点（`test_steps` 个），计算相邻点之间的距离
   - 若设置了相机，将距离转换为屏幕空间像素尺寸
   - 根据 `dicing_rate` 计算所需段数：`tmin = ceil(Lsum / dicing_rate)` 和 `tmax = ceil((N-1) * Lmax / dicing_rate)`
   - 若 `tmax - tmin > split_threshold`，标记为 `DSPLIT_NON_UNIFORM`，需要进一步分裂

2. **递归分裂**:
   - 四边形面：沿 U 或 V 方向分裂为两个子四边形，选择较长的轴优先分裂
   - 三角形面：选择最长的需分裂边，从中点到对角顶点创建新边，分裂为两个子三角形
   - N 边形面：先在中心点处分解为 N 个四边形，然后对每个四边形递归分裂
   - 递归终止条件：所有边的细分因子 T 为正值（不再需要分裂），或达到最大深度 `DSPLIT_MAX_DEPTH`

3. **水密性保证**:
   - 共享边在哈希集合 `edges` 中只存储一次，两侧面片引用同一个 `SubEdge`
   - 分裂时对共享边使用一致的中点和子边分配
   - 边的 `own_vertex` / `own_edge` 标记确保每个顶点和边的属性只被一个子面片负责设置

### De Casteljau 双三次曲面求值

`BicubicPatch` 使用经典的 De Casteljau 算法在参数坐标 (u, v) 处求值：
- 先沿 u 方向对 4 行控制点各做一次三次 De Casteljau 求值（4 次调用 `decasteljau_cubic`）
- 再沿 v 方向对 4 个结果点做一次三次 De Casteljau 求值
- 同时计算偏导数 du、dv，法线 N 由 `cross(du, dv)` 归一化得到

### EdgeDice 网格切分

采用 DX11 风格的 EdgeDice 方式（参见 ARB_tessellation_shader OpenGL 扩展 Section 2.X.2）：

1. **边界顶点设置**: 先并行设置所有子面片的边界顶点坐标
2. **内部网格生成**:
   - 四边形：生成 `(Mu-1) x (Mv-1)` 的内部网格，然后与四条边界缝合
   - 三角形：生成 `M*(M-1)/2` 个内部网格顶点，然后与三条边界缝合
3. **缝合算法**: 内部网格边缘与外部边之间通过选择最短对角线方向创建三角形，确保三角形形状合理
4. **并行化**: 使用 TBB 的 `parallel_for`，以每 8 个子面片为一组进行并行处理

### OpenSubdiv Catmull-Clark 集成

当编译时启用 `WITH_OPENSUBDIV` 且网格使用 Catmull-Clark 细分模式时：

1. **拓扑精炼**: 通过 `TopologyRefinerFactory<OsdMesh>` 特化，将 Cycles 的 Mesh 拓扑注入 OpenSubdiv
   - 支持边折痕（Edge Crease）和顶点折痕（Vertex Crease），使用 Pixar 历史标准的 10.0 最大折痕权重
   - 支持多种 FVar 线性插值模式和边界插值模式
2. **自适应精炼**: 使用 `RefineAdaptive`，最大隔离层级为 3
3. **Patch 表构建**: 使用 Gregory Basis 端帽（End Cap）处理非正则顶点
4. **FVar 合并**: 对面变量属性（如 UV），将同一顶点处值相同的角点合并，构建 FVar 拓扑通道
5. **Patch 求值**: `OsdPatch::eval()` 通过 `PatchMap::FindPatch` 定位 patch 后，使用 `EvaluateBasis` 计算权重并加权求和

### 属性插值策略

| 属性类型 | 线性模式 | Catmull-Clark 模式 |
|---------|---------|-------------------|
| 顶点属性（位置类） | 双线性/N 边形加权插值 | OpenSubdiv `PrimvarRefiner` 精炼后 patch 求值 |
| 顶点属性（其他） | 双线性/N 边形加权插值 | 线性插值（同线性模式） |
| 角点属性（UV 等） | 线性角点插值 | OpenSubdiv FVar 精炼后 patch 求值 |
| 角点属性（非 FVar） | 线性角点插值 | 线性角点插值 |
| 面属性 | 直接复制到子三角形 | 直接复制到子三角形 |

对 `uchar4`（字节颜色）类型，插值过程在 `float4` 精度下进行，最终转换回字节格式，以避免精度损失。

## 参见

- `src/scene/mesh_subdivision.cpp` — 细分曲面的主要调用入口，驱动整个 subd 流程
- `src/scene/mesh.h` — `Mesh` 类定义，包含 `SubdFace`、`SubdEdgeCrease` 等细分相关数据结构
- `src/scene/geometry.cpp` — 几何体管理，触发细分曲面的更新
- DiagSplit 论文: "DiagSplit: Parallel, Crack-free, Adaptive Tessellation for Micro-polygon Rendering"
- ARB_tessellation_shader OpenGL 扩展（Section 2.X.2）— EdgeDice 算法的参考来源
- [OpenSubdiv 文档](https://graphics.pixar.com/opensubdiv/docs/intro.html) — Catmull-Clark 细分的底层实现库
