# embree.h / embree.cpp - Intel Embree 光线追踪后端集成

## 概述

本文件实现了 Cycles 渲染器与 Intel Embree 光线追踪库的集成,提供基于 Embree 的硬件加速层次包围体(BVH)构建和遍历。`BVHEmbree` 类继承自 `BVH` 基类,将 Cycles 的场景几何体(三角形、曲线、点云)转换为 Embree 原生格式,支持运动模糊、对象实例化、SYCL GPU 加速(Embree GPU)以及 BVH 动态重拟合。

## 类与结构体

### BVHEmbree
- **继承**: `BVH`（定义于 `bvh/bvh.h`）
- **功能**: 管理 Embree RTCScene 的创建、几何体添加、BVH 构建和重拟合。
- **关键成员**:
  - `scene` (`RTCScene`) — Embree 场景句柄,包含所有已添加的几何体
  - `rtc_device` (`RTCDevice`) — Embree 设备句柄
  - `rtc_device_is_sycl` (`bool`) — 是否为 SYCL GPU 设备
  - `build_quality` (`RTCBuildQuality`) — 构建质量级别
- **关键方法**:
  - `build(Progress &progress, Stats *stats, RTCDevice rtc_device, bool rtc_device_is_sycl)` — 完整构建 Embree BVH
  - `refit(Progress &progress)` — 更新顶点缓冲区并重新提交场景
  - `offload_scenes_to_gpu(const vector<RTCScene> &scenes)` — 将 BVH 数据迁移到 GPU（仅 Embree GPU + SYCL）
  - `get_error_string(RTCError error_code)` — 获取 Embree 错误码对应的字符串
  - `add_object(Object *ob, int i)` — 添加对象几何体(分发到 triangles/curves/points)
  - `add_instance(Object *ob, int i)` — 添加对象实例(使用 Embree 实例几何体)
  - `add_triangles(const Object *ob, const Mesh *mesh, int i)` — 添加三角形网格
  - `add_curves(const Object *ob, const Hair *hair, int i)` — 添加毛发曲线
  - `add_points(const Object *ob, const PointCloud *pointcloud, int i)` — 添加点云
  - `set_tri_vertex_buffer(RTCGeometry, const Mesh *, bool update)` — 设置/更新三角形顶点缓冲区
  - `set_curve_vertex_buffer(RTCGeometry, const Hair *, bool update)` — 设置/更新曲线控制点缓冲区
  - `set_point_vertex_buffer(RTCGeometry, const PointCloud *, bool update)` — 设置/更新点云顶点缓冲区

## 核心函数

### `BVHEmbree::build()`
1. 设置 Embree 设备错误回调和内存监控回调
2. 创建新的 `RTCScene`,配置场景标志:
   - 动态 BVH 时启用 `RTC_SCENE_FLAG_DYNAMIC`
   - 紧凑结构时启用 `RTC_SCENE_FLAG_COMPACT`
   - 始终启用 `RTC_SCENE_FLAG_ROBUST` 和 `RTC_SCENE_FLAG_FILTER_FUNCTION_IN_ARGUMENTS`
3. 根据 BVH 类型设置构建质量:动态->LOW,空间分割->HIGH,默认->MEDIUM
4. 遍历所有对象,顶层 BVH 中实例化对象使用 `add_instance()`,其他使用 `add_object()`
5. 设置进度回调并提交场景(`rtcCommitScene`)

### `BVHEmbree::add_instance()`
1. 创建 `RTC_GEOMETRY_TYPE_INSTANCE` 类型几何体
2. 设置实例化场景为子对象的 Embree 场景
3. 对运动模糊对象,将运动变换分解为四元数旋转+缩放+平移+错切,通过 `rtcSetGeometryTransformQuaternion` 设置
4. 对静态对象,直接设置 3x4 行主序变换矩阵
5. 以 `i * 2` 作为几何体 ID 附加到场景

### `BVHEmbree::add_triangles()`
1. 创建 `RTC_GEOMETRY_TYPE_TRIANGLE` 类型几何体
2. CPU 设备使用 `rtcSetSharedGeometryBuffer` 共享 Cycles 的索引缓冲区
3. SYCL GPU 设备创建新的 Embree 缓冲区并拷贝数据
4. 调用 `set_tri_vertex_buffer` 设置顶点数据
5. 以 `i * 2` 作为几何体 ID 附加到场景

### `BVHEmbree::add_curves()`
1. 根据曲线形状选择 Embree 几何体类型:
   - `CURVE_THICK_LINEAR` -> `RTC_GEOMETRY_TYPE_ROUND_LINEAR_CURVE`
   - `CURVE_RIBBON` -> `RTC_GEOMETRY_TYPE_FLAT_CATMULL_ROM_CURVE`
   - 默认 -> `RTC_GEOMETRY_TYPE_ROUND_CATMULL_ROM_CURVE`
2. 创建索引缓冲区,Catmull-Rom 样条需要额外偏移(每条曲线首尾各加一个重复控制点)
3. 以 `i * 2 + 1` 作为几何体 ID 附加到场景(与三角形错开)

### 模板函数 `pack_motion_verts<T>()`
将毛发运动控制点打包为 `float4`(x, y, z, radius)格式。对 Catmull-Rom 样条,在每条曲线的首尾复制控制点以满足 Embree 的端点要求。

## 依赖关系

- **内部头文件**:
  - `bvh/bvh.h` — 基类 `BVH`
  - `bvh/params.h` — `BVHParams`
  - `scene/hair.h`、`scene/mesh.h`、`scene/object.h`、`scene/pointcloud.h` — 场景几何体
  - `util/log.h`、`util/progress.h`、`util/stats.h` — 日志、进度、内存统计
- **外部依赖**:
  - `embree4/rtcore.h`、`embree4/rtcore_scene.h`、`embree4/rtcore_geometry.h` — Embree 4 API
- **被引用**:
  - `bvh/bvh.cpp` — 工厂方法中创建 `BVHEmbree` 实例（条件编译 `WITH_EMBREE`）
  - `device/cpu/device_impl.cpp` — CPU 设备中使用 Embree BVH 进行射线遍历

## 实现细节 / 关键算法

### Cycles ID 到 Embree ID 的映射
由于 Embree 不允许同一几何体同时包含三角形和曲线,Cycles 将每个对象的 Embree ID 设为 `对象索引 * 2`,曲线几何体使用 `对象索引 * 2 + 1`,以此区分。

### CPU vs SYCL GPU 缓冲区管理
CPU 路径使用 `rtcSetSharedGeometryBuffer` 直接共享 Cycles 的内存,避免数据拷贝。SYCL GPU 路径(Embree GPU)由于不能使用主机指针,需要通过 `rtcSetNewGeometryBuffer`(或 Embree 4.4+ 的 `rtcSetNewGeometryBufferHostDevice`)分配设备缓冲区并拷贝数据。GPU 端的 `float3` 需要转换为 `packed_float3` 格式。

### 内存监控
通过 `rtcSetDeviceMemoryMonitorFunction` 注册内存回调,将 Embree 的内存分配/释放通知转发到 Cycles 的 `Stats` 统计系统。在 `Stats` 指针尚未可用时,使用原子变量 `unaccounted_mem` 暂存。

### 运动模糊支持
对三角形和曲线,通过 `rtcSetGeometryTimeStepCount` 设置运动步数,每个步对应一个独立的顶点缓冲区。中间时间步使用当前帧的顶点数据,其他步从 `ATTR_STD_MOTION_VERTEX_POSITION` 属性获取。实例的运动变换使用四元数分解以确保插值正确性。

### GPU BVH 预取
`offload_scenes_to_gpu()` 通过设置 `RTC_SCENE_FLAG_PREFETCH_USM_SHARED_ON_GPU` 标志并重新提交场景,强制将 BVH 数据迁移到 GPU,确保渲染性能优先。

## 关联文件

- `bvh/bvh.h` / `bvh/bvh.cpp` — BVH 基类和工厂方法
- `bvh/params.h` — BVH 参数
- `device/cpu/device_impl.cpp` — CPU 设备集成 Embree 进行射线遍历
- `scene/mesh.h`、`scene/hair.h`、`scene/pointcloud.h` — 场景几何体数据
