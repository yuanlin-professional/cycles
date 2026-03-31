# particles.h / particles.cpp — 粒子系统

## 概述

`particles.h` 和 `particles.cpp` 实现了 Cycles 的粒子系统数据管理，包括粒子数据结构 `Particle`、粒子系统节点 `ParticleSystem` 和粒子系统管理器 `ParticleSystemManager`。粒子数据从 Blender 导入后存储在 `ParticleSystem` 中，管理器负责将粒子信息打包上传到渲染设备供着色器访问（如粒子信息节点）。

## 类与结构体

### Particle

- **功能**: 单个粒子的数据结构体
- **关键成员**:
  - `index` (`int`) — 粒子索引
  - `age` (`float`) — 粒子年龄
  - `lifetime` (`float`) — 粒子生命周期
  - `location` (`float3`) — 粒子位置
  - `rotation` (`float4`) — 粒子旋转（四元数）
  - `size` (`float`) — 粒子大小
  - `velocity` (`float3`) — 粒子速度
  - `angular_velocity` (`float3`) — 粒子角速度

### ParticleSystem

- **继承**: `Node`
- **功能**: 粒子系统节点，包含一组粒子
- **关键成员**:
  - `particles` (`array<Particle>`) — 粒子数组
- **关键方法**:
  - `tag_update()` — 标记粒子系统需要更新

### ParticleSystemManager

- **功能**: 管理所有粒子系统的设备数据同步
- **关键成员**:
  - `need_update_` (`bool`) — 是否需要更新
- **关键方法**:
  - `device_update()` — 设备更新入口，调用 `device_update_particles()`
  - `device_update_particles()` — 将粒子数据打包为 `KernelParticle` 数组并上传
  - `device_free()` — 释放设备内存
  - `tag_update()` — 标记需要更新
  - `need_update()` — 查询是否需要更新

## 核心函数

### 粒子数据打包

`device_update_particles()` 流程：
1. 统计所有粒子系统的粒子总数，添加 1 个虚拟粒子（索引 0）以避免无效查找
2. 分配 `KernelParticle` 设备数组
3. 遍历所有粒子系统的粒子，将 `Particle` 结构体中的数据填入 `KernelParticle`
4. 位置/速度/角速度转为 `float4` 格式上传
5. 调用 `copy_to_device()` 传输到 GPU

## 依赖关系

- **内部头文件**: `util/array.h`, `util/types.h`, `graph/node.h`
- **被引用**: `scene/object.h`（Object 引用 ParticleSystem）, `scene/object.cpp`, `scene/scene.cpp`

## 实现细节 / 关键算法

- **虚拟粒子**: 在设备数组起始位置插入一个全零的虚拟粒子（索引 0），当着色器中的粒子信息节点在无粒子数据的对象上使用时，返回安全的默认值。
- **粒子偏移**: `ObjectManager` 中通过 `particle_offset` 映射表将每个粒子系统映射到全局粒子数组中的起始偏移（偏移从 1 开始，因为 0 是虚拟粒子）。对象的 `particle_index` 加上所属粒子系统的偏移得到全局粒子索引。
- **数据量**: 粒子数据仅在着色器中通过粒子信息节点查询时使用，不参与几何体的光线追踪。

## 关联文件

- `scene/object.h` — `Object` 通过 `particle_system` 和 `particle_index` 引用粒子
- `scene/object.cpp` — `ObjectManager::device_update_transforms()` 中计算粒子偏移
- `scene/scene.h` — `Scene` 持有 `particle_systems` 列表和 `particle_system_manager`
