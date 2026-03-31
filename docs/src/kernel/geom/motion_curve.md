# motion_curve.h - 运动模糊曲线关键帧插值

## 概述
本文件实现了运动模糊(motion blur)场景下曲线图元的控制点插值。运动曲线以常规曲线形式存储，附加额外的运动步骤位置和半径数据。通过在两个相邻时间步之间进行线性插值，计算出给定光线时间点处的曲线控制点位置和半径。额外的曲线控制点作为 `ATTR_STD_MOTION_VERTEX_POSITION` 属性存储。

## 核心函数

### motion_curve_keys_for_step_linear()
- **签名**: `ccl_device_inline void motion_curve_keys_for_step_linear(KernelGlobals kg, int offset, const int numverts, const int numsteps, int step, const int k0, const int k1, float4 keys[2])`
- **功能**: 获取指定时间步的两个曲线控制点(线性段)。中心步使用常规 `curve_keys` 数据，其他步从 `attributes_float4` 运动属性数组中读取。

### motion_curve_keys_linear()
- **签名**: `ccl_device_inline void motion_curve_keys_linear(KernelGlobals kg, const int object, const float time, const int k0, const int k1, float4 keys[2])`
- **功能**: 计算给定时间的两个运动曲线控制点（线性插值版本）。根据时间确定相邻的两个时间步及插值因子 t，分别获取两步的控制点后进行线性插值。用于曲线厚度等仅需两个点的计算场景。

### motion_curve_keys_for_step()
- **签名**: `ccl_device_inline void motion_curve_keys_for_step(KernelGlobals kg, int offset, const int numverts, const int numsteps, int step, const int k0, const int k1, const int k2, const int k3, float4 keys[4])`
- **功能**: 获取指定时间步的四个曲线控制点(Catmull-Rom 段)。处理逻辑与线性版本类似，但同时读取四个控制点。

### motion_curve_keys()
- **签名**: `ccl_device_inline void motion_curve_keys(KernelGlobals kg, const int object, const float time, const int k0, const int k1, const int k2, const int k3, float4 keys[4])`
- **功能**: 计算给定时间的四个运动曲线控制点。这是曲线相交测试和着色设置的主要入口。根据时间参数在两个相邻运动步之间插值，返回完整的 Catmull-Rom 四控制点数据。

## 依赖关系
- **内部头文件**: `kernel/globals.h`, `kernel/bvh/util.h`
- **被引用**: `geom/curve_intersect.h`, `geom/curve.h`

## 实现细节 / 关键算法

### 时间步映射
- 运动步数(numsteps)表示中心帧前后各有多少步，总步数为 `maxstep = numsteps * 2`
- 时间 time 在 [0, 1] 范围内映射到 [0, maxstep] 的步索引
- `step = min((int)(time * maxstep), maxstep - 1)` 确定当前步
- `t = time * maxstep - step` 为步间插值因子

### 中心步特殊处理
中心步(step == numsteps)的数据存储在常规的 `curve_keys` 数组中而非运动属性数组。运动属性数组不包含中心步数据，因此当 step > numsteps 时需要将步索引减一以正确映射到属性数组偏移。

### 数据布局
运动属性中每个步存储 numverts 个 float4 值(位置xyz + 半径w)，偏移计算为 `offset + step * numverts + key_index`。

整个文件被 `__HAIR__` 预处理宏保护。

## 关联文件
- `kernel/geom/curve_intersect.h` - 曲线相交测试中调用本文件的插值函数
- `kernel/geom/curve.h` - 曲线厚度计算中调用线性插值版本
- `kernel/bvh/util.h` - BVH 工具函数，提供 `intersection_find_attribute()`
