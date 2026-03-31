# motion_point.h - 运动模糊点云关键帧插值

## 概述
本文件实现了运动模糊(motion blur)场景下点云(point cloud)图元的位置和半径插值。与运动曲线类似，运动点以常规点形式存储，附加额外的运动步骤位置数据作为 `ATTR_STD_MOTION_VERTEX_POSITION` 属性。在给定光线时间处，通过两个相邻时间步之间的线性插值计算点的位置和半径。

## 核心函数

### motion_point_for_step()
- **签名**: `ccl_device_inline float4 motion_point_for_step(KernelGlobals kg, int offset, const int numverts, const int numsteps, int step, const int prim)`
- **功能**: 获取指定时间步的单个点云图元数据(位置 xyz + 半径 w)。中心步(step == numsteps)从常规 `points` 数组读取，其他步从运动属性数组 `attributes_float4` 中读取。

### motion_point()
- **签名**: `ccl_device_inline float4 motion_point(KernelGlobals kg, const int object, const int prim, const float time)`
- **功能**: 计算给定时间点处运动点云图元的位置和半径。确定相邻两个时间步及插值因子 t，获取两步的数据后线性插值返回 float4 结果(xyz = 位置, w = 半径)。

## 依赖关系
- **内部头文件**: `kernel/globals.h`, `kernel/bvh/util.h`
- **被引用**: `geom/point_intersect.h`, `geom/point.h`

## 实现细节 / 关键算法

### 时间步映射
与 `motion_curve.h` 相同的时间步映射策略：
- `maxstep = numsteps * 2`，时间 time 映射到 [0, maxstep]
- 中心步数据存储在常规数组中，运动属性数组跳过中心步

### 数据格式
每个点以 float4 格式存储，xyz 分量为位置坐标，w 分量为半径。运动插值同时作用于位置和半径。

整个文件被 `__POINTCLOUD__` 预处理宏保护。

## 关联文件
- `kernel/geom/point_intersect.h` - 点云相交测试中调用本文件获取运动位置
- `kernel/geom/point.h` - 点云着色数据中调用本文件获取运动位置
- `kernel/bvh/util.h` - 提供 `intersection_find_attribute()` 属性查找
