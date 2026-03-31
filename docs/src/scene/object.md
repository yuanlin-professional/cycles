# object.h / object.cpp — 场景对象与对象管理器

## 概述

`object.h` 和 `object.cpp` 定义了 Cycles 场景中的对象实例 `Object` 及其管理器 `ObjectManager`。`Object` 将几何体（`Geometry`）实例化到场景中，携带变换矩阵、可见性标志、材质属性、粒子系统引用等。`ObjectManager` 负责将所有对象数据打包上传到渲染设备，处理运动模糊变换分解和静态变换优化。

## 类与结构体

### Object

- **继承**: `Node`
- **功能**: 场景中的一个对象实例，关联几何体并定义变换、材质和可见性属性
- **关键成员**:
  - `geometry` (`Geometry*`) — 引用的几何体
  - `tfm` (`Transform`) — 物体到世界空间的变换矩阵
  - `bounds` (`BoundBox`) — 世界空间包围盒
  - `random_id` (`uint`) — 随机 ID（用于着色器中的随机值）
  - `pass_id` (`int`) — 通道 ID
  - `color` (`float3`) — 对象颜色
  - `alpha` (`float`) — 对象透明度
  - `asset_name` (`ustring`) — 资产名称（Cryptomatte 用）
  - `visibility` (`uint`) — 可见性标志位
  - `motion` (`array<Transform>`) — 运动模糊变换数组
  - `hide_on_missing_motion` (`bool`) — 缺少运动数据时隐藏对象（用于粒子消失）
  - `use_holdout` (`bool`) — 是否作为遮罩体
  - `is_shadow_catcher` (`bool`) — 是否为阴影捕捉器
  - `shadow_terminator_shading_offset` / `shadow_terminator_geometry_offset` (`float`) — 阴影终止线偏移
  - `is_caustics_caster` / `is_caustics_receiver` (`bool`) — 焦散投射/接收
  - `dupli_generated` (`float3`) — Dupli 生成坐标
  - `dupli_uv` (`float2`) — Dupli UV 坐标
  - `particle_system` (`ParticleSystem*`) — 关联的粒子系统
  - `particle_index` (`int`) — 粒子索引
  - `ao_distance` (`float`) — 环境光遮蔽距离
  - `lightgroup` (`ustring`) — 灯光组名称
  - `receiver_light_set` / `light_set_membership` — 灯光链接接收集/成员
  - `blocker_shadow_set` / `shadow_set_membership` — 阴影链接阻挡集/成员
  - `intersects_volume` (`bool`) — 是否与体积对象相交（设备更新时设置）
  - `index` (`int`) — 在 `scene->objects` 中的索引
  - `attr_map_offset` (`size_t`) — 对象属性映射偏移
- **关键方法**:
  - `tag_update()` — 标记更新，联动几何/灯光/体积管理器
  - `compute_bounds()` — 计算世界空间包围盒（支持运动模糊）
  - `apply_transform()` — 将对象变换烘焙到几何体顶点中
  - `use_motion()` — 是否有运动模糊
  - `motion_time()` / `motion_step()` — 运动步与归一化时间的转换
  - `update_motion()` — 验证和清理运动数据
  - `is_traceable()` — 是否可追踪（非灯光、有效包围盒）
  - `visibility_for_tracing()` — 结合阴影捕捉器的最终可见性
  - `compute_volume_step_size()` — 从体素网格和着色器计算体积步长
  - `usable_as_light()` — 是否可作为面光源（有发光着色器且可见）
  - `has_light_linking()` / `has_shadow_linking()` — 检查灯光/阴影链接参与

### ObjectManager

- **功能**: 管理所有对象的设备数据上传和变换处理
- **关键成员**:
  - `update_flags` (`uint32_t`) — 更新标志位
  - `need_flags_update` (`bool`) — 是否需要标志更新
- **更新标志枚举**:
  - `PARTICLE_MODIFIED`, `GEOMETRY_MANAGER`, `MOTION_BLUR_MODIFIED`
  - `OBJECT_ADDED` / `OBJECT_REMOVED` / `OBJECT_MODIFIED`
  - `HOLDOUT_MODIFIED`, `TRANSFORM_MODIFIED`, `VISIBILITY_MODIFIED`
- **关键方法**:
  - `device_update()` — 主更新：分配索引、上传变换数据
  - `device_update_transforms()` — 并行处理所有对象变换
  - `device_update_prim_offsets()` — 更新 MetalRT/HIPRT 的图元偏移
  - `device_update_flags()` — 更新对象标志（体积、阴影捕捉器、体积相交等）
  - `device_update_geom_offsets()` — 更新属性映射和顶点数偏移
  - `apply_static_transforms()` — 对单用户几何体应用静态变换优化
  - `get_cryptomatte_objects()` / `get_cryptomatte_assets()` — 生成 Cryptomatte 清单
  - `tag_update()` — 标记更新，联动几何/灯光/积分器管理器

### UpdateObjectTransformState（object.cpp 内部）

- **功能**: 设备变换更新的全局状态，供并行任务共享
- **关键成员**: `need_motion`, `particle_offset`, `motion_offset`, 设备数组指针, 场景特征标志

## 核心函数

### 变换上传（并行）

`device_update_transforms()` 使用 TBB `parallel_for` 以 32 对象为粒度并行处理：
1. 计算逆变换矩阵
2. 填充 `KernelObject` 结构（变换、颜色、ID、粒子等）
3. 处理运动模糊：`MOTION_PASS` 模式存储前/后变换，`MOTION_BLUR` 模式分解为四元数+平移
4. 设置对象标志（负缩放、顶点运动、体积运动等）

### 静态变换优化

`apply_static_transforms()` 对仅被一个对象引用、无 BSSRDF、无真实位移的几何体，将变换直接烘焙到顶点坐标中，避免内核中的逐光线变换计算。曲线和点云仅在均匀缩放时应用。

### 运动模糊包围盒

`Object::compute_bounds()` 在运动模糊模式下，将运动数组分解为四元数+平移，在 [0,1] 区间内以 1/128 步长采样插值变换矩阵，取所有步的变换包围盒并集。

### 体积步长计算

`compute_volume_step_size()` 逻辑：
1. 遍历着色器，取最小 `volume_step_rate`
2. 遍历体素属性，从元数据和变换计算体素大小
3. 用户指定步长优先，否则自动检测
4. 无体素数据时退回到包围盒 1/10

## 依赖关系

- **内部头文件**: `graph/node.h`, `scene/geometry.h`, `scene/particles.h`, `scene/scene.h`, `util/array.h`, `util/boundbox.h`, `util/transform.h`
- **被引用**: `scene/geometry.cpp`, `scene/volume.cpp`, `scene/light.cpp`, `scene/scene.cpp`, `scene/mesh_displace.cpp`, `scene/hair.cpp`, `scene/camera.cpp` 等

## 实现细节 / 关键算法

- **运动变换分解**: 使用 `transform_motion_decompose()` 将 Transform 矩阵分解为 `DecomposedTransform`（四元数旋转+平移+缩放），确保插值在旋转空间中正确。
- **阴影捕捉器可见性**: `visibility_for_tracing()` 使用 `SHADOW_CATCHER_OBJECT_VISIBILITY` 宏，确保阴影捕捉器对特定光线类型可见。
- **Cryptomatte**: 使用 MurmurHash3 对对象名和资产名进行哈希，生成 `float` 标识符用于后期合成。
- **MAX_MOTION_STEPS**: 限制为 129（Embree 的限制）。

## 关联文件

- `scene/geometry.h` — 几何体基类
- `scene/particles.h` — 粒子系统
- `scene/scene.h` — 场景管理
- `scene/volume.h` — 体积管理器（变换变更时联动）
- `scene/light.h` — 灯光管理器（发光网格联动）
