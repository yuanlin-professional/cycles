# camera.h / camera.cpp - 摄像机参数管理与变换计算

## 概述

本文件定义了 Cycles 渲染器的摄像机系统。`Camera` 类继承自 `Node`，封装了所有摄像机参数，包括投影类型（透视、正交、全景、自定义）、景深、运动模糊、卷帘快门、立体视觉等。该类负责计算从光栅空间到世界空间的各种变换矩阵，以及将参数打包传输到渲染设备内核。此外还支持通过 OSL 脚本实现自定义摄像机投影。

## 类与结构体

### OSLCameraParamQuery

- **功能**: 纯虚基类，定义 OSL 自定义摄像机参数查询接口。
- **关键方法**:
  - `get_float(ustring, vector<float>&)` — 查询浮点参数
  - `get_int(ustring, vector<int>&)` — 查询整数参数
  - `get_string(ustring, string&)` — 查询字符串参数

### Camera

- **继承**: `Node`（节点系统基类）
- **功能**: 管理摄像机的所有参数、计算变换矩阵链、处理运动模糊和景深、同步数据到设备内核。

#### 投影类型（`camera_type`）
| 类型 | 说明 |
|------|------|
| `CAMERA_PERSPECTIVE` | 透视投影 |
| `CAMERA_ORTHOGRAPHIC` | 正交投影 |
| `CAMERA_PANORAMA` | 全景投影 |
| `CAMERA_CUSTOM` | 自定义 OSL 脚本投影 |

#### 卷帘快门类型（`RollingShutterType`）
| 类型 | 说明 |
|------|------|
| `ROLLING_SHUTTER_NONE` | 无卷帘快门效果 |
| `ROLLING_SHUTTER_TOP` | 从顶部到底部扫描 |

#### 立体视觉类型（`StereoEye`）
| 类型 | 说明 |
|------|------|
| `STEREO_NONE` | 非立体 |
| `STEREO_LEFT` | 左眼 |
| `STEREO_RIGHT` | 右眼 |

- **关键成员（参数类）**:
  - `camera_type` — 摄像机投影类型
  - `fov` — 视场角
  - `matrix` — 摄像机到世界的变换矩阵
  - `nearclip` / `farclip` — 近/远裁剪面
  - `focaldistance` / `aperturesize` / `blades` / `bladesrotation` — 景深参数
  - `shuttertime` / `motion_position` / `shutter_curve` — 运动模糊参数
  - `rolling_shutter_type` / `rolling_shutter_duration` — 卷帘快门参数
  - `panorama_type` / `fisheye_fov` / `fisheye_lens` — 全景摄像机参数
  - `stereo_eye` / `interocular_distance` / `convergence_distance` — 立体视觉参数
  - `sensorwidth` / `sensorheight` — 传感器尺寸
  - `full_width` / `full_height` — 完整渲染分辨率
  - `viewplane` / `border` / `viewport_camera_border` — 视平面与边界区域
  - `aperture_ratio` — 变形镜头散景比例
  - `motion` — 运动模糊变换数组
- **关键成员（计算结果）**:
  - `screentoworld` / `rastertoworld` / `ndctoworld` — 各空间到世界空间的投影变换
  - `worldtoraster` / `worldtoscreen` / `worldtondc` / `worldtocamera` — 世界空间到各空间的变换
  - `rastertocamera` / `full_rastertocamera` — 光栅到摄像机空间变换
  - `dx` / `dy` / `full_dx` / `full_dy` — 像素微分
  - `frustum_right_normal` / `frustum_top_normal` / `frustum_left_normal` / `frustum_bottom_normal` — 视锥体平面法线
  - `kernel_camera` — 设备端内核摄像机数据副本
  - `kernel_camera_motion` — 分解后的运动模糊变换数组
  - `script_name` / `script_params` — OSL 自定义摄像机脚本及参数
- **关键方法**:
  - `update(Scene*)` — 根据当前参数计算所有变换矩阵和内核数据
  - `device_update(Device*, DeviceScene*, Scene*)` — 将数据同步到设备（含快门曲线查找表）
  - `device_update_volume(Device*, DeviceScene*, Scene*)` — 检测摄像机是否在体积内部
  - `device_free(Device*, DeviceScene*, Scene*)` — 释放设备端资源
  - `compute_auto_viewplane()` — 根据宽高比自动计算视平面
  - `viewplane_bounds_get()` — 获取视平面在世界空间的包围盒
  - `world_to_raster_size(float3)` — 计算世界空间点对应的像素尺寸（用于自适应细分）
  - `motion_time(int)` / `motion_step(float)` — 运动模糊时间步转换
  - `use_motion()` — 是否使用运动模糊
  - `set_screen_size(int, int)` — 设置渲染分辨率
  - `set_osl_camera()` / `clear_osl_camera()` — 设置/清除 OSL 自定义摄像机
  - `get_kernel_features()` — 返回所需的内核特性标志

## 核心函数

- `shutter_curve_eval()` — 评估快门曲线函数，通过线性插值采样用户定义的快门响应曲线
- `Camera::transform_full_raster_to_world()` — 将完整光栅坐标转换到世界空间点（内部辅助函数，用于包围盒计算）

## 依赖关系

- **内部头文件**:
  - `kernel/types.h` — `KernelCamera`、`CameraType`、`PanoramaType` 等内核类型
  - `graph/node.h` — 节点系统基类
  - `util/array.h`、`util/boundbox.h`、`util/projection.h`、`util/transform.h`
  - `scene/mesh.h`、`scene/object.h`、`scene/osl.h`、`scene/scene.h`、`scene/tables.h`（cpp 中引用）
  - `kernel/camera/camera.h` — 内核摄像机采样函数（用于 `world_to_raster_size`）
- **被引用**: `scene/film.cpp`、`scene/integrator.cpp`、`scene/object.cpp`、`scene/geometry.cpp`、`scene/geometry_bvh.cpp`、`scene/geometry_mesh.cpp`、`scene/geometry_attributes.cpp`、`scene/shader.cpp`、`scene/osl.cpp`、`scene/scene.cpp`、`session/session.cpp`、`subd/split.cpp`、`subd/dice.cpp`、`hydra/camera.cpp`、`app/` 等约 18 个文件

## 实现细节 / 关键算法

1. **变换矩阵链计算** (`update`): 从视平面参数和投影类型出发，构建完整的坐标空间变换链：NDC -> 光栅 -> 屏幕 -> 摄像机 -> 世界，以及反向链。对于透视投影使用透视投影矩阵，正交投影使用正交投影矩阵，全景和自定义投影使用单位矩阵。

2. **运动模糊处理**: 支持两种运动模型——运动向量通道（`MOTION_PASS`）和运动模糊渲染（`MOTION_BLUR`）。前者计算起止时刻的透视矩阵，后者将运动变换分解为四元数和平移分量存储到设备。透视运动模糊额外支持 FOV 随时间变化。

3. **世界空间像素尺寸** (`world_to_raster_size`): 对于透视摄像机，利用射线微分计算世界空间中某点的像素尺寸。对于屏幕外的点，根据距视锥体边缘的距离按 `offscreen_dicing_scale` 缩放，以减少屏幕外几何的细分密度。

4. **体积内检测** (`device_update_volume`): 并行检查所有含体积的对象包围盒是否与视平面包围盒相交，以确定摄像机是否在体积内部。自定义摄像机始终认为在体积内。

5. **快门曲线**: 使用逆 CDF（累积分布函数）将用户定义的快门响应曲线转换为查找表，用于运动模糊时间采样的重要性采样。

6. **自定义摄像机 OSL**: 支持加载 OSL 着色器作为自定义摄像机投影。通过 `OSLCameraParamQuery` 接口从主机获取参数值，支持 int、float 和 string 类型。由于自适应细分的限制，自定义摄像机回退使用透视投影计算 `full_rastertocamera`。

## 关联文件

- `scene/film.h` / `scene/film.cpp` — 胶片参数影响摄像机更新
- `scene/integrator.h` — 运动模糊设置
- `scene/tables.h` — 快门曲线查找表管理
- `scene/osl.h` — OSL 着色系统（自定义摄像机）
- `kernel/camera/camera.h` — 内核端摄像机采样函数
- `kernel/types.h` — `KernelCamera` 设备端数据结构
