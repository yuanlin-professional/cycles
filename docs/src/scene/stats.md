# stats.h / stats.cpp - 渲染统计与性能分析系统

## 概述

本文件实现了 Cycles 渲染器的统计信息收集与报告系统。提供了多种统计条目类型（大小统计、时间统计、采样统计、采样-计数对统计），以及面向网格、图像、内核和场景更新的专用统计容器。所有统计类都支持生成格式化的人类可读报告。

## 类与结构体

### NamedSizeEntry
- **功能**: 带名称的大小统计条目
- **关键成员**: `name` — 名称, `size` — 大小值

### NamedTimeEntry
- **功能**: 带名称的时间统计条目
- **关键成员**: `name` — 名称, `time` — 时间值（秒）

### NamedSizeStats
- **功能**: 大小统计条目容器，自动跟踪总大小
- **关键成员**: `total_size` — 所有条目总大小, `entries` — 条目列表
- **关键方法**:
  - `add_entry()` — 添加条目并累加总大小
  - `full_report()` — 生成按大小降序排列的格式化报告

### NamedTimeStats
- **功能**: 时间统计条目容器，自动跟踪总时间
- **关键成员**: `total_time` — 所有条目总时间, `entries` — 条目列表
- **关键方法**:
  - `add_entry()` — 添加条目并累加总时间
  - `full_report()` — 生成按时间降序排列的格式化报告
  - `clear()` — 清除所有条目和总时间

### NamedNestedSampleStats
- **功能**: 嵌套采样统计，支持树形结构（用于内核性能分析）
- **关键成员**:
  - `name` — 事件名称
  - `self_samples` — 自身采样数
  - `sum_samples` — 自身加所有子条目的采样总数
  - `entries` — 子条目列表
- **关键方法**:
  - `add_entry()` — 添加子条目并返回其引用
  - `update_sum()` — 递归更新 `sum_samples`
  - `full_report()` — 生成缩进的树形报告（显示总占比和自身占比）

### NamedSampleCountPair
- **功能**: 采样数-命中数对，用于着色器和物体性能统计
- **关键成员**: `name` — 名称, `samples` — 采样数, `hits` — 命中数

### NamedSampleCountStats
- **功能**: 采样-计数对统计容器，支持按名称合并
- **关键成员**: `entries` — `unordered_map<ustring, NamedSampleCountPair>` 条目映射
- **关键方法**:
  - `add()` — 添加或合并条目
  - `full_report()` — 生成按采样数降序排列的报告，显示相对耗时

### MeshStats
- **功能**: 网格/几何体内存统计
- **关键成员**: `geometry` — 几何体大小统计（`NamedSizeStats`）

### ImageStats
- **功能**: 图像/纹理内存统计
- **关键成员**: `textures` — 纹理大小统计（`NamedSizeStats`）

### RenderStats
- **功能**: 渲染过程综合统计，汇总网格、图像、内核和着色器/物体统计
- **关键成员**:
  - `has_profiling` — 是否有内核性能数据（仅 CPU 渲染可用）
  - `mesh` — 网格统计
  - `image` — 图像统计
  - `kernel` — 内核嵌套采样统计
  - `shaders` — 着色器耗时统计
  - `objects` — 物体耗时统计
- **关键方法**:
  - `collect_profiling()` — 从 Profiler 收集内核采样信息（光线设置、交叉测试、表面/体积/阴影着色等分类）
  - `full_report()` — 生成完整报告

### UpdateTimeStats
- **功能**: 更新时间统计包装器
- **关键成员**: `times` — `NamedTimeStats` 实例

### SceneUpdateStats
- **功能**: 场景更新各阶段的时间统计
- **关键成员**: `geometry`, `image`, `light`, `object`, `background`, `bake`, `camera`, `film`, `integrator`, `osl`, `particles`, `scene`, `svm`, `tables`, `procedurals` — 各子系统的 UpdateTimeStats
- **关键方法**:
  - `full_report()` — 生成场景更新完整时间报告
  - `clear()` — 清除所有子系统统计

## 核心函数

- `namedSizeEntryComparator()` — 按大小降序排序比较器
- `namedTimeEntryComparator()` — 按时间降序排序比较器
- `namedTimeSampleEntryComparator()` — 按采样总数降序排序比较器
- `namedSampleCountPairComparator()` — 按采样数降序排序比较器

## 依赖关系

- **内部头文件**: `scene/scene.h`, `util/string.h`, `util/vector.h`
- **cpp 额外引用**: `scene/object.h`, `util/algorithm.h`
- **被引用**: `scene/image.cpp`, `scene/tables.cpp`, `scene/procedural.cpp`, `scene/geometry.cpp`, `scene/light.cpp`, `scene/object.cpp`, `scene/film.cpp`, `scene/background.cpp`, `scene/bake.cpp`, `scene/camera.cpp`, `scene/integrator.cpp`, `scene/osl.cpp`, `scene/particles.cpp`, `scene/svm.cpp`, `session/session.h`

## 实现细节 / 关键算法

1. **报告格式**: 所有 `full_report()` 使用 `kIndentNumSpaces = 2` 的缩进，条目按相关度量降序排列显示。
2. **内核性能分析**: `collect_profiling()` 构建四层树形结构：总渲染时间 -> 光线/着色大类 -> 着色子类（设置、评估、直接光、间接光等）。采样数乘以 0.001 转换为近似秒数。
3. **相对耗时**: `NamedSampleCountStats::full_report()` 计算每命中平均采样数，然后显示每个条目相对于平均值的比率作为"相对耗时"。
4. **线程安全**: 统计数据的收集发生在各管理器的 `device_update()` 回调中，通过 `scoped_callback_timer` 自动计时。

## 关联文件

- `scene/scene.h/.cpp` — 场景管理（持有 `SceneUpdateStats`，在各更新步骤中记录时间）
- `session/session.h` — 渲染会话（持有 `RenderStats`）
