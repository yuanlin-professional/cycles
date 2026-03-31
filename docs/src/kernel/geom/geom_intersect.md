# geom_intersect.h - 几何相交通用工具

## 概述
本文件提供了各种几何类型相交测试的通用工具函数。主要包含局部相交(local intersection)的命中记录索引管理，用于次表面散射(subsurface scattering)等需要在同一对象内进行多次命中记录的场景。

## 核心函数

### local_intersect_get_record_index()
- **签名**: `ccl_device_forceinline int local_intersect_get_record_index(LocalIntersection *local_isect, const float isect_t, uint *lcg_state, const int max_hits)`
- **功能**: 为局部相交测试管理命中记录的索引分配。支持两种模式：
  - **多命中模式**(`lcg_state` 非空)：记录最多 `max_hits` 个交点。当命中数超过上限时，使用蓄水池抽样(reservoir sampling)随机替换已有记录，确保每个交点被选中的概率相等。去重检查避免记录相同距离的重复交点。
  - **最近命中模式**(`lcg_state` 为空)：仅记录最近的一个交点。如果新交点距离更远则忽略。
  - 返回值 -1 表示该交点应被忽略，非负值表示应写入 `local_isect->hits` 的索引位置。

## 依赖关系
- **内部头文件**: `kernel/globals.h`, `kernel/sample/lcg.h`
- **被引用**: `geom/triangle_intersect.h`, `geom/motion_triangle_intersect.h`

## 实现细节 / 关键算法

### 蓄水池抽样(Reservoir Sampling)
当局部相交的命中数超过 `max_hits` 上限时，采用蓄水池抽样算法：
- 递增总命中计数器 `num_hits`
- 通过线性同余生成器(LCG) `lcg_step_uint()` 生成随机数
- 对 `num_hits` 取模得到候选索引，若候选索引 >= max_hits 则跳过（返回 -1），否则替换该位置的记录
- 该算法保证在流式处理中，每个命中记录被选中的概率均为 max_hits / num_hits

此机制被 `__BVH_LOCAL__` 预处理宏保护，仅在启用局部层次包围体(BVH)遍历功能时编译。

## 关联文件
- `kernel/geom/triangle_intersect.h` - 三角形局部相交调用本函数
- `kernel/geom/motion_triangle_intersect.h` - 运动三角形局部相交调用本函数
- `kernel/sample/lcg.h` - 线性同余随机数生成器
