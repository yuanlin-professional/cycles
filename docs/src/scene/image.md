# image.h / image.cpp - 图像资源管理与加载系统

## 概述

本文件实现了 Cycles 渲染器的核心图像管理系统，负责所有 2D 纹理图像和 3D 体积图像的加载、存储与设备端同步。系统采用基于槽位（slot）的引用计数机制管理图像生命周期，支持多种数据格式（float、half、byte、ushort、NanoVDB 等）和 UDIM 瓦片纹理。通过抽象的 `ImageLoader` 基类，支持从文件、内存或程序化来源加载图像。

## 类与结构体

### ImageParams
- **功能**: 图像加载参数配置
- **关键成员**:
  - `animated` — 是否为动画纹理
  - `interpolation` — 插值类型（线性、最近邻等）
  - `extension` — 纹理扩展模式（裁剪、重复等）
  - `alpha_type` — Alpha 通道处理方式（自动、忽略、预乘等）
  - `colorspace` — 色彩空间标识（默认为 raw）
  - `frame` — 动画帧号

### ImageMetaData
- **功能**: 图像元数据，在像素加载前即可获取的信息
- **关键成员**:
  - `width`, `height` — 图像尺寸
  - `channels` — 通道数
  - `type` — 数据类型（IMAGE_DATA_TYPE_FLOAT4 等）
  - `colorspace` — 色彩空间
  - `use_transform_3d`, `transform_3d` — 3D 体积图像的可选变换矩阵
  - `compress_as_srgb` — 是否以 sRGB 压缩存储
- **关键方法**:
  - `is_float()` — 判断是否为浮点类型
  - `detect_colorspace()` — 自动检测并转换色彩空间，决定是否需要格式升级（如 byte 升级为 half 以支持 HDR 色彩空间转换）

### ImageDeviceFeatures
- **功能**: 描述设备端支持的图像特性
- **关键成员**: `has_nanovdb` — 是否支持 NanoVDB 体积数据格式

### ImageLoader
- **功能**: 图像加载器抽象基类，可派生以支持不同数据源
- **关键方法**:
  - `load_metadata()` — 快速加载元数据（纯虚函数）
  - `load_pixels()` — 加载实际像素数据（纯虚函数）
  - `name()` — 返回用于日志的名称（纯虚函数）
  - `osl_filepath()` — 可选，返回 OSL 纹理缓存路径
  - `get_tile_number()` — 可选，返回 UDIM 瓦片编号
  - `cleanup()` — 释放加载过程中分配的临时内存
  - `equals()` — 比较两个加载器是否引用相同图像（用于去重）
  - `is_vdb_loader()` — 是否为 VDB 加载器（RTTI 替代方案）

### ImageHandle
- **功能**: 图像访问句柄，采用引用计数管理。多个着色器节点可共享同一图像。
- **关键成员**:
  - `slots` — 在 ImageManager 中对应的槽位索引列表
  - `is_tiled` — 是否为瓦片纹理
  - `manager` — 所属的 ImageManager 指针
- **关键方法**:
  - `clear()` — 释放引用，减少所有槽位的用户计数
  - `metadata()` — 获取元数据（懒加载）
  - `svm_slot()` — 获取 SVM 着色器虚拟机槽位索引
  - `get_svm_slots()` — 获取所有瓦片的 SVM 槽位信息（打包为 int4）
  - `image_memory()` — 获取设备端纹理内存指针
  - `vdb_loader()` — 获取关联的 VDB 加载器（如有）

### ImageManager
- **功能**: 图像管理器，管理场景中所有图像的加载、存储与设备同步
- **关键成员**:
  - `images` — 图像槽位数组（`vector<unique_ptr<Image>>`）
  - `features` — 设备特性描述
  - `osl_texture_system` — OSL 纹理系统句柄
  - `animation_frame` — 当前动画帧
  - `device_mutex`, `images_mutex` — 线程同步互斥锁
- **关键方法**:
  - `add_image()` — 添加图像（4 个重载：文件名、文件名+瓦片、自定义加载器、多加载器），返回 ImageHandle
  - `device_update()` — 将所有需要更新的图像上传到设备，使用 TaskPool 并行加载
  - `device_update_slot()` — 更新单个槽位
  - `device_free()` — 释放所有设备端图像内存
  - `device_load_builtin()` — 仅加载内建（Blender depsgraph）图像
  - `set_osl_texture_system()` — 设置 OSL 纹理系统
  - `set_animation_frame_update()` — 更新动画帧，返回是否需要重新加载
  - `collect_statistics()` — 收集内存统计信息
- **内部结构 Image**:
  - `params` — 图像参数
  - `metadata` — 元数据
  - `loader` — 加载器实例
  - `need_metadata`, `need_load` — 加载状态标记
  - `builtin` — 是否为 Blender 内建图像
  - `mem` — 设备端纹理内存
  - `users` — 引用计数

## 核心函数

- `name_from_type()` — 内部辅助函数，将 ImageDataType 枚举转换为字符串
- `image_associate_alpha()` — 判断是否需要对图像执行 alpha 预乘
- `file_load_image<FileFormat, StorageType>()` — 模板函数，执行像素加载、通道转换（1/2/3 通道转 RGBA）、色彩空间转换、NaN 清理和纹理缩放

## 依赖关系

- **内部头文件**: `device/memory.h`, `scene/colorspace.h`, `util/string.h`, `util/thread.h`, `util/transform.h`, `util/unique_ptr.h`, `util/vector.h`
- **cpp 额外引用**: `device/device.h`, `scene/image_oiio.h`, `scene/image_vdb.h`, `scene/scene.h`, `scene/stats.h`, `util/image.h`, `util/image_impl.h`, `util/progress.h`, `util/task.h`, `util/texture.h`; 条件编译: `OSL/oslexec.h`
- **被引用**: `scene/scene.h`, `scene/image_oiio.h`, `scene/image_sky.h`, `scene/image_vdb.h`, `scene/shader_nodes.h`, `scene/shader_nodes.cpp`, `scene/attribute.h`, `scene/attribute.cpp`, `kernel/osl/services.h`, `hydra/field.h`

## 实现细节 / 关键算法

1. **槽位去重**: `add_image_slot()` 在分配新槽位前，遍历已有图像查找相同的加载器+参数组合，找到则增加引用计数直接复用。
2. **延迟释放**: 用户计数降为 0 时不立即释放图像，而是在下一次 `device_update()` 中统一清理，避免着色器重编译时频繁重载。
3. **并行加载**: `device_update()` 使用 `TaskPool` 并行加载所有待更新的图像到设备。
4. **通道转换**: `file_load_image()` 将 1/2/3 通道图像统一转换为 4 通道 RGBA 格式（或保持单通道），内核仅支持 1 通道和 4 通道。
5. **纹理缩放**: 当 `texture_limit > 0` 且图像尺寸超限时，使用 `util_image_resize_pixels()` 按 0.5 的幂次缩小。
6. **加载失败处理**: 加载失败时创建 1x1 粉红色像素作为占位符（TEX_IMAGE_MISSING_*）。

## 关联文件

- `scene/image_oiio.h/.cpp` — OIIO 文件加载器实现
- `scene/image_sky.h/.cpp` — 天空纹理程序化加载器
- `scene/image_vdb.h/.cpp` — OpenVDB/NanoVDB 体积加载器
- `scene/colorspace.h/.cpp` — 色彩空间转换
- `scene/stats.h/.cpp` — 渲染统计
- `scene/scene.h/.cpp` — 场景管理（持有 ImageManager）
