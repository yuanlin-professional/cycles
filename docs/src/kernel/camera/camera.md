# camera.h - 相机光线生成与采样内核函数

## 概述
本文件是 Cycles 渲染器相机子系统的核心，负责**根据光栅坐标和随机采样参数生成从相机出发的光线 (Ray)**。它统一封装了透视、正交、全景和自定义四种相机类型的光线生成逻辑，是路径追踪积分器的起点。文件同时提供了相机位置查询、距离计算和世界坐标到 NDC 坐标的转换等实用工具函数。

设计上采用分发模式：顶层函数 `camera_sample()` 根据相机类型调度到对应的专用采样函数，每个函数独立处理景深模拟、运动模糊插值和光线微分计算。

## 类与结构体
本文件无自定义结构体定义。核心操作的数据结构包括：
- **`KernelCamera`** (来自 `kernel/types.h`) — 存储所有相机参数（变换矩阵、光圈、焦距、裁剪距离等）
- **`Ray`** — 光线结构，包含原点 `P`、方向 `D`、时间 `time`、范围 `tmin`/`tmax` 以及光线微分 `dP`/`dD`

## 枚举与常量
本文件引用以下枚举（定义于 `kernel/types.h`）：

| 枚举值 | 说明 |
|---|---|
| `CAMERA_PERSPECTIVE` | 透视相机 |
| `CAMERA_ORTHOGRAPHIC` | 正交相机 |
| `CAMERA_PANORAMA` | 全景相机 |
| `CAMERA_CUSTOM` | 自定义相机（OSL 着色器驱动） |

引用的常量：
- `FILTER_TABLE_SIZE` — 像素滤波器查找表大小
- `SHUTTER_TABLE_SIZE` — 快门时间查找表大小

## 核心函数

### camera_sample_aperture()
- **签名**: `ccl_device float2 camera_sample_aperture(ccl_constant KernelCamera *cam, const float2 rand)`
- **功能**: 在相机光圈上采样一个点，用于景深 (DOF) 模拟。当 `blades == 0` 时在圆盘上均匀采样；否则在正多边形上采样（模拟光圈叶片形状）。采样结果会乘以 `inv_aperture_ratio` 以支持变形镜头 (anamorphic lens) 的椭圆形散景效果。

### camera_sample_perspective()
- **签名**: `ccl_device Spectrum camera_sample_perspective(KernelGlobals kg, const float2 raster_xy, const float2 rand_lens, ccl_private Ray *ray)`
- **功能**: 为**透视相机**生成光线。处理流程：
  1. 通过 `rastertocamera` 变换将光栅坐标转换到相机空间
  2. 如有透视运动模糊 (`have_perspective_motion`)，在前后两个投影矩阵间插值
  3. 若光圈大小 > 0，执行景深模拟：在光圈上采样偏移原点，重新计算指向焦平面的方向
  4. 通过 `cameratoworld` 变换（含运动插值）将光线转换到世界空间
  5. 处理立体渲染偏移（若启用）
  6. 计算光线微分 `dP`/`dD`（用于纹理抗锯齿）
  7. 应用近裁剪面和远裁剪面
- **返回值**: `one_spectrum()`，表示相机灵敏度为 1（均匀灵敏度）

### camera_sample_orthographic()
- **签名**: `ccl_device Spectrum camera_sample_orthographic(KernelGlobals kg, const float2 raster_xy, const float2 rand_lens, ccl_private Ray *ray)`
- **功能**: 为**正交相机**生成光线。与透视相机的关键区别：
  - 初始光线方向固定为 `(0, 0, 1)` （相机空间 Z 轴正方向）
  - 景深处理时需要额外计算光线到达近裁剪面的位置偏移
  - 光线微分中 `dP` 非零（因为正交投影下相邻像素的光线原点不同），而 `dD` 为零（平行光线）
  - 不支持立体渲染

### camera_sample_to_ray()
- **签名**: `ccl_device_inline void camera_sample_to_ray(ccl_constant KernelCamera *cam, const ccl_global DecomposedTransform *cam_motion, float3 P, float3 D, ..., ccl_private Ray *ray)`
- **功能**: **全景和自定义相机的公共光线变换函数**。将相机空间中的光线原点和方向变换到世界空间，处理运动模糊插值、球面立体声变换和光线微分计算。被 `camera_sample_panorama()` 和 `camera_sample_custom()` 共享调用。

### camera_sample_custom()
- **签名**: `ccl_device_inline Spectrum camera_sample_custom(KernelGlobals kg, ccl_constant KernelCamera *cam, const ccl_global DecomposedTransform *cam_motion, const float2 raster, const float2 rand_lens, ccl_private Ray *ray)`
- **功能**: 为**自定义相机**生成光线。通过 OSL 着色器 (`osl_eval_camera()`) 计算光线位置、方向和透过率。需要 `WITH_OSL` 编译宏支持，否则返回零光谱（无效光线）。OSL 着色器返回零透过率时也表示采样失败。

### camera_panorama_direction()
- **签名**: `ccl_device_inline float3 camera_panorama_direction(ccl_constant KernelCamera *cam, const float x, const float y)`
- **功能**: 辅助函数。将光栅坐标经 `rastertocamera` 变换后，调用 `panorama_to_direction()` 获取全景相机的光线方向。

### camera_sample_panorama()
- **签名**: `ccl_device_inline Spectrum camera_sample_panorama(ccl_constant KernelCamera *cam, const ccl_global DecomposedTransform *cam_motion, const float2 raster, const float2 rand_lens, ccl_private Ray *ray)`
- **功能**: 为**全景相机**生成光线。处理流程：
  1. 通过 `camera_panorama_direction()` 获取光线方向
  2. 方向为零向量时表示采样失败（如鱼眼投影越界），返回零光谱
  3. 执行景深模拟：计算焦平面交点，构建与方向垂直的正交坐标系 (U, V)，在该平面上偏移光线原点
  4. 调用 `camera_sample_to_ray()` 完成世界空间变换和微分计算
  5. 同时计算相邻像素方向用于光线微分

### camera_sample()
- **签名**: `ccl_device_inline Spectrum camera_sample(KernelGlobals kg, const int x, const int y, const float2 filter_uv, const float time, const float2 lens_uv, ccl_private Ray *ray)`
- **功能**: **相机光线采样的统一入口**。积分器通过此函数生成所有类型的相机光线。处理流程：
  1. **像素滤波**: 使用预计算的滤波器查找表对像素坐标施加抖动偏移
  2. **运动模糊时间**: 通过快门查找表将随机数映射为快门时间；支持滚动快门效果（自上而下扫描线采集），通过 `rolling_shutter_duration` 在运动模糊和滚动快门之间线性插值
  3. **类型分发**: 根据 `kernel_data.cam.type` 调用对应的专用采样函数
- **返回值**: 相机灵敏度光谱，作为路径追踪吞吐量的初始值

### camera_position()
- **签名**: `ccl_device_inline float3 camera_position(KernelGlobals kg)`
- **功能**: 从 `cameratoworld` 变换矩阵中提取相机在世界空间中的位置（矩阵第四列）。

### camera_distance()
- **签名**: `ccl_device_inline float camera_distance(KernelGlobals kg, const float3 P)`
- **功能**: 计算世界空间中点 `P` 到相机的距离。正交相机计算到相机平面的投影距离（点积），透视相机和其他类型计算欧氏距离。

### camera_z_depth()
- **签名**: `ccl_device_inline float camera_z_depth(KernelGlobals kg, const float3 P)`
- **功能**: 计算点 `P` 的 Z 深度值。透视/正交相机通过 `worldtocamera` 变换获取相机空间 Z 坐标；全景/自定义相机退化为欧氏距离。

### camera_direction_from_point()
- **签名**: `ccl_device_inline float3 camera_direction_from_point(KernelGlobals kg, const float3 P)`
- **功能**: 计算从点 `P` 指向相机的单位方向向量。正交相机返回负 Z 轴方向（相机平面法线），其他相机返回归一化的位置差向量。

### camera_world_to_ndc()
- **签名**: `ccl_device_inline float3 camera_world_to_ndc(KernelGlobals kg, ccl_private ShaderData *sd, float3 P)`
- **功能**: 将世界空间点转换为 NDC (归一化设备坐标)。透视/正交相机使用 `worldtondc` 投影变换；全景相机先变换到相机空间再调用 `direction_to_panorama()` 进行逆投影；自定义相机暂退回相机坐标（尚无逆映射实现）。对于背景物体 (`PRIM_NONE`)，透视相机需先加上相机位置偏移。

## 依赖关系
- **内部头文件**:
  - `kernel/globals.h` — 全局内核数据访问宏 (`kernel_data`, `kernel_data_array`)
  - `kernel/camera/projection.h` — 全景投影函数和球面立体声变换
  - `kernel/sample/mapping.h` — 采样函数（`sample_uniform_disk`, `regular_polygon_sample`）
  - `kernel/util/differential.h` — 光线微分工具（`differential_make_compact`, `differential_zero_compact`）
  - `kernel/util/lookup_table.h` — 查找表读取（`lookup_table_read`）
  - `kernel/osl/camera.h` — OSL 自定义相机着色器（条件编译 `WITH_OSL`）
- **被引用**:
  - `src/kernel/integrator/init_from_camera.h` — 积分器相机光线初始化
  - `src/kernel/integrator/init_from_bake.h` — 烘焙相机光线初始化
  - `src/kernel/film/data_passes.h` — 胶片/成像平面数据通道
  - `src/kernel/svm/tex_coord.h` — 纹理坐标节点
  - `src/kernel/osl/services_gpu.h` — OSL GPU 服务
  - `src/kernel/osl/services.cpp` — OSL CPU 服务
  - `src/scene/camera.cpp` — 场景层相机对象

## 实现细节 / 关键算法

### 景深模拟 (Depth of Field)
三种支持景深的相机类型（透视、正交、全景）共享相同的物理模型：
1. 在光圈形状上采样一个偏移点（由 `camera_sample_aperture()` 完成）
2. 计算原始光线与焦平面的交点 `Pfocus`
3. 将光线原点偏移到光圈采样点，方向重新指向焦平面交点
4. 焦平面距离由 `focaldistance` 控制

全景相机的景深实现较为特殊：需要先构建与光线方向垂直的局部坐标系 (U, V)，然后在该平面上偏移，以保持球面投影的正确性。

### 透视运动模糊 (Perspective Motion)
`camera_sample_perspective()` 支持透视矩阵的运动模糊。通过在 `perspective_pre`（时间 0）和 `rastertocamera`（时间 0.5）以及 `perspective_post`（时间 1）三个投影矩阵之间分段线性插值来实现。这种方法直接插值投影后的坐标而非视场角，计算简单且视觉效果良好。

### 滚动快门 (Rolling Shutter)
`camera_sample()` 中实现了简化的滚动快门模型：
- 画面顶部扫描线时间为 0.0，底部为 1.0
- `rolling_shutter_duration` 控制运动模糊与滚动快门效果的混合比例
- `duration = 0` 时为纯滚动快门，`duration = 1` 时为纯运动模糊

### 光线微分 (Ray Differentials)
光线微分用于纹理过滤（抗锯齿），在 `__RAY_DIFFERENTIALS__` 宏保护下计算：
- **透视相机**: `dP` 为零（共同原点），`dD` 通过相邻像素方向差分计算
- **正交相机**: `dP` 通过相机空间 dx/dy 偏移计算，`dD` 为零（平行光线）
- **全景/自定义相机**: 通过独立计算中心像素及相邻像素的完整光线来获取 `dP` 和 `dD` 差分
- 立体渲染模式下，所有微分从零开始重新计算以避免景深影响

### 近远裁剪面
所有相机类型在最后阶段应用裁剪：
- 光线原点沿方向前移 `nearclip` 距离
- `tmax` 设为 `cliplength`（透视相机还需乘以 `z_inv` 校正因子以补偿非归一化方向）

## 关联文件
- `src/kernel/camera/projection.h` — 全景投影数学函数库
- `src/kernel/integrator/init_from_camera.h` — 积分器从相机发射路径的起点，直接调用 `camera_sample()`
- `src/kernel/types.h` — `KernelCamera` 结构体与相机类型枚举定义
- `src/kernel/sample/mapping.h` — 光圈采样所使用的圆盘和多边形采样函数
- `src/kernel/util/differential.h` — 光线微分的紧凑表示与工具函数
- `src/kernel/osl/camera.h` — OSL 自定义相机着色器接口
- `src/scene/camera.cpp` — 场景层相机对象，负责设置 `KernelCamera` 中的参数
