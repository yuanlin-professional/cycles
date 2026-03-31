# texture.h - 纹理类型定义与纹理信息结构体

## 概述
本文件定义了 Cycles 渲染器中与纹理相关的基础枚举类型和数据结构，包括纹理插值方式、图像数据类型、Alpha 处理模式、纹理扩展（重复/镜像/裁切）方式，以及纹理元数据描述结构体 `TextureInfo`。这些定义在 CPU 和 GPU 设备上通用，是纹理采样管线的基础。

## 类与结构体

### `struct TextureInfo`
纹理元数据描述结构体：
| 成员 | 类型 | 说明 |
|------|------|------|
| `data` | `uint64_t` | 指针、偏移或设备纹理句柄（取决于后端） |
| `data_type` | `uint` | 图像数据类型（`ImageDataType` 枚举） |
| `interpolation` | `uint` | 插值模式 |
| `extension` | `uint` | 扩展模式 |
| `width` / `height` | `uint` | 纹理尺寸 |
| `use_transform_3d` | `uint` | 是否使用 3D 变换 |
| `transform_3d` | `Transform` | 3D 纹理变换矩阵 |

### 枚举定义

#### `InterpolationType` - 纹理插值方式
| 值 | 说明 |
|------|------|
| `INTERPOLATION_LINEAR` | 线性插值 |
| `INTERPOLATION_CLOSEST` | 最近邻插值 |
| `INTERPOLATION_CUBIC` | 三次插值 |
| `INTERPOLATION_SMART` | 智能插值 |

#### `ImageDataType` - 图像数据类型
| 值 | 说明 |
|------|------|
| `IMAGE_DATA_TYPE_FLOAT4` / `FLOAT` | 32 位浮点（4/1 通道） |
| `IMAGE_DATA_TYPE_BYTE4` / `BYTE` | 8 位无符号整型 |
| `IMAGE_DATA_TYPE_HALF4` / `HALF` | 16 位半精度浮点 |
| `IMAGE_DATA_TYPE_USHORT4` / `USHORT` | 16 位无符号整型 |
| `IMAGE_DATA_TYPE_NANOVDB_*` | NanoVDB 体积纹理类型（FLOAT/FLOAT3/FLOAT4/FPN/FP16/EMPTY） |

#### `ImageAlphaType` - Alpha 通道处理方式
| 值 | 说明 |
|------|------|
| `IMAGE_ALPHA_UNASSOCIATED` | 非预乘 Alpha |
| `IMAGE_ALPHA_ASSOCIATED` | 预乘 Alpha |
| `IMAGE_ALPHA_CHANNEL_PACKED` | Alpha 打包在通道中 |
| `IMAGE_ALPHA_IGNORE` | 忽略 Alpha |
| `IMAGE_ALPHA_AUTO` | 自动检测 |

#### `ExtensionType` - 纹理扩展方式
| 值 | 说明 |
|------|------|
| `EXTENSION_REPEAT` | 平铺重复 |
| `EXTENSION_EXTEND` | 边缘像素延伸 |
| `EXTENSION_CLIP` | 裁切（外部透明） |
| `EXTENSION_MIRROR` | 镜像翻转 |

### 辅助函数
| 函数 | 说明 |
|------|------|
| `bool is_nanovdb_type(int type)` | 判断数据类型是否为 NanoVDB 体积纹理类型 |

## 核心函数
无独立函数（除 `is_nanovdb_type`），本文件以类型定义为主。

### 缺失纹理默认颜色
当纹理文件缺失时使用的默认颜色为品红色 `(R=1, G=0, B=1, A=1)`，便于视觉识别。

## 依赖关系
- **内部头文件**: `util/defines.h`, `util/transform.h`
- **被引用**: `kernel/` 下的纹理采样内核, `scene/image.h`, `device/` 下的设备纹理管理

## 关联文件
- `util/transform.h` - `Transform` 结构体定义
- `util/image.h` - 图像 I/O 工具
- `util/nanovdb.h` - NanoVDB 体积纹理支持
