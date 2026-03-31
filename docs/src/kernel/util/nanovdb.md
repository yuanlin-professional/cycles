# nanovdb.h - NanoVDB 体积数据内核端访问接口

## 概述
本文件是 OpenVDB NanoVDB 库的精简提取版本，仅包含内核端（GPU/CPU）访问体积网格数据所需的最小代码。原始 NanoVDB 头文件由于缺少 Metal 地址空间修饰符而不兼容 Metal 后端，因此 Cycles 维护了这个经过适配的版本。文件实现了 NanoVDB 的完整层次化稀疏数据结构——从 Grid（网格）、Tree（树）、RootNode（根节点）、InternalNode（内部节点）到 LeafNode（叶节点）的完整访问链，以及用于高效体素查询的 ReadAccessor 和 CachedReadAccessor。

## 类与结构体

### `Mask<LOG2DIM>`
位掩码模板结构体，用于标记内部节点和叶节点中每个子条目是否活跃（active）。使用 64 位字数组存储，提供 `isOff()` 方法检查特定位是否关闭。

### `Grid<TreeT>`
NanoVDB 网格的顶层数据结构，包含元数据（魔数、版本、校验和、网格名称、包围盒、体素尺寸等）以及对 Tree 的访问接口。对齐到 `NANOVDB_DATA_ALIGNMENT`（32 字节）。

### `Tree<RootT>`
树结构，存储各层节点的偏移量和数量。通过 `root()` 方法提供对根节点的访问，使用 `mNodeOffset` 数组中的偏移量进行指针运算定位。

### `RootNode<ChildT>`
根节点，使用线性查找的 Tile 表来定位子节点。每个 Tile 存储一个键值（由坐标编码而来）、子节点偏移、状态标志和值。提供 `probeTile()` 方法通过线性搜索查找坐标对应的 Tile，以及 `getChild()` 方法获取子节点指针。

### `InternalNode<ChildT, Log2Dim>`
内部节点模板，使用固定大小的 Tile 表和两个位掩码（值掩码 `mValueMask` 和子节点掩码 `mChildMask`）进行快速查找。通过 `CoordToOffset()` 静态方法将三维坐标转换为表内偏移。

### `LeafData<ValueT, LOG2DIM>`
叶节点数据的通用模板，直接存储值数组 `mValues`。

### `LeafData<Fp16, LOG2DIM>`
Fp16 量化叶节点的模板特化。使用 16 位整数编码值，通过量化因子 `mQuantum` 和最小值 `mMinimum` 进行反量化。

### `LeafData<FpN, LOG2DIM>`
FpN（N 位浮点）量化叶节点的模板特化。支持可变位宽压缩（由 `mFlags` 的高 3 位指定），通过位操作解码。

### `LeafNode<BuildT, Log2Dim>`
叶节点模板，封装 `LeafData` 并提供坐标到偏移量的转换以及值查询接口。默认 `Log2Dim = 3`，即每个叶节点覆盖 8x8x8 = 512 个体素。

### `ReadAccessor<BuildT>`
简单的只读访问器，从根节点开始逐级向下查找以获取指定坐标的值。无缓存机制，每次查询都从根节点开始。

### `CachedReadAccessor<BuildT>`
带三级缓存的只读访问器，缓存最近访问的 Upper、Lower 和 Leaf 节点。查询时先检查缓存，命中则直接从缓存节点继续向下查找，可显著减少重复的树遍历开销。

## 枚举与常量

- **`NANOVDB_DATA_ALIGNMENT`** — 数据对齐值，设为 32 字节
- **`NANOVDB_USE_SINGLE_ROOT_KEY`** — 启用单 64 位整数作为根节点查找键（而非 Coord 坐标），将三维坐标编码为一个 uint64

## 核心函数

### PtrAdd()
- **签名**: `template<typename DstT, typename SrcT> const ccl_global DstT *PtrAdd(const ccl_global SrcT *p, int64_t offset)`
- **功能**: 通用指针偏移工具，将源指针按字节偏移后转换为目标类型指针。用于在 NanoVDB 的紧凑内存布局中进行节点定位。

### RootNode::CoordToKey()
- **签名**: `static ccl_device_inline_method uint64_t CoordToKey(const Coord ijk)`
- **功能**: 将三维整数坐标编码为 64 位查找键。坐标值先右移 `ChildT::TOTAL` 位以对齐到根节点的空间分辨率，然后组合到单个 uint64 中（z: bit0-20, y: bit21-41, x: bit42-62）。

### InternalNode::CoordToOffset()
- **签名**: `static ccl_device_inline_method uint32_t CoordToOffset(const Coord ijk)`
- **功能**: 将三维坐标转换为内部节点 Tile 表的线性偏移量。通过掩码和移位操作提取坐标中属于当前层级的位。

### LeafNode::CoordToOffset()
- **签名**: `static ccl_device_inline_method uint32_t CoordToOffset(const Coord ijk)`
- **功能**: 将三维坐标转换为叶节点值数组的线性偏移量。

## 依赖关系
- **内部头文件**:
  - `util/defines.h` — 编译器宏定义
  - `util/math_int3.h` — int3 数学运算
  - `util/types_base.h` — 基础类型
  - `util/types_int3.h` — int3 类型定义
- **被引用**:
  - `kernel/util/texture_3d.h` — 3D 纹理插值中使用 NanoVDB 访问器读取体积数据
  - `kernel/device/metal/context_begin.h` — Metal 后端上下文设置
  - `kernel/device/oneapi/context_begin.h` — oneAPI 后端上下文设置

## 实现细节 / 关键算法

### 层次化稀疏数据结构
NanoVDB 使用四级树结构存储稀疏体积数据：
1. **RootNode** — 根节点，使用线性 Tile 表，键为 64 位编码坐标
2. **NanoUpper** (InternalNode, Log2Dim=5) — 上层内部节点，覆盖 4096^3 体素
3. **NanoLower** (InternalNode, Log2Dim=4) — 下层内部节点，覆盖 128^3 体素
4. **NanoLeaf** (LeafNode, Log2Dim=3) — 叶节点，覆盖 8^3 = 512 体素

每级内部节点通过位掩码 `mChildMask` 区分子节点和常量值 Tile，实现高效的稀疏存储。

### 缓存访问器
`CachedReadAccessor` 为每一级节点（Leaf、Lower、Upper）维护一个缓存槽，存储最近访问的节点指针和对应的坐标键。查询时通过检查坐标是否落在缓存节点的覆盖范围内（使用掩码比较）来判断缓存命中。这对于空间相干的体积查询（如沿光线步进）非常有效。

### 量化叶节点
支持两种压缩格式：
- **Fp16**：每个值用 16 位无符号整数存储，通过 `value = code * quantum + minimum` 反量化
- **FpN**：可变位宽（由 flags 的高 3 位控制），通过位操作从紧凑的 uint32 数组中提取值

### Metal 兼容性
所有结构体成员和方法均添加了 `ccl_global` 地址空间修饰符，以兼容 Metal 着色语言的地址空间要求。`double` 类型的成员（如 WorldBBox、VoxelSize）被替换为等大小的 `uint8_t` 数组以避免 Metal 中不支持 double 的问题。

## 关联文件
- `src/kernel/util/texture_3d.h` — 使用本文件的 NanoVDB 访问器进行 3D 纹理插值
- `src/kernel/geom/volume.h` — 体积几何处理
- `src/scene/image_vdb.cpp` — 主机端 VDB 数据加载和 NanoVDB 转换
