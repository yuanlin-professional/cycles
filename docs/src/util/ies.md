# ies.h / ies.cpp - IES光域文件解析器

## 概述
本文件实现了 IES（Illuminating Engineering Society）光域文件的解析和处理。IES 文件描述了灯具的光强分布，Cycles 通过解析该文件将光度学数据（坎德拉/sr）转换为辐射度量（瓦特/sr），并统一到 Type C 坐标系以供渲染内核使用。支持 IES 标准中的三种光度类型（Type A/B/C），处理对称性镜像和角度归一化。

## 类与结构体

### `class IESFile`
IES 文件解析与处理的核心类。

#### 公有方法
| 方法 | 说明 |
|------|------|
| `bool load(const string &ies)` | 加载并处理 IES 文件内容字符串，失败时自动清理 |
| `void clear()` | 清除所有已解析数据 |
| `int packed_size()` | 返回打包后的浮点数组所需大小 |
| `void pack(float *data)` | 将数据打包为连续浮点数组，格式：[h_count, v_count, h_angles..., v_angles..., intensity...] |

#### 保护成员
| 成员 | 说明 |
|------|------|
| `vector<float> v_angles` | 垂直角度数组（球坐标的 phi） |
| `vector<float> h_angles` | 水平角度数组（球坐标的 theta，0-360 度） |
| `vector<vector<float>> intensity` | 光强二维数组，每行对应一个水平角度的垂直分布 |
| `enum IESType type` | 光度类型：TYPE_A(3), TYPE_B(2), TYPE_C(1) |

#### 保护方法
| 方法 | 说明 |
|------|------|
| `bool parse(const string &ies)` | 解析 IES 文本，提取角度和光强数据 |
| `bool process()` | 将解析数据转换为统一的 Type C 格式 |
| `void process_type_a/b/c()` | 各光度类型的具体转换逻辑 |

### `class IESTextParser`（内部类，cpp 文件）
IES 文本解析辅助类：
- 将逗号替换为空格
- 定位 `TILT=` 标记
- 提供 `get_double()` 和 `get_long()` 解析方法，含错误状态追踪

## 核心函数

### 光度学转换
解析过程中，光强值从坎德拉（lumen/sr）转换为瓦特/sr：
- 使用 D65 标准光源的光视效能 177.83 lm/W
- 转换因子 = `4*pi / 177.83 = 0.0706650768394`

### Type A/B/C 处理
- **Type C**（最常见）：处理 90-270 度偏移、单象限/双象限对称镜像、补全缺失的 360 度条目
- **Type B**：先转置水平/垂直角度，再根据 0-90 或 -90-90 范围进行镜像和偏移
- **Type A**：垂直角度偏移 90 度，水平角度从 -90~90 映射到 270~90（Type C 格式）

### 最终处理
所有角度从度数转换为弧度（乘以 `M_PI_F / 180`）。

## 依赖关系
- **内部头文件**: `util/string.h`, `util/vector.h`, `util/math.h`（cpp）
- **被引用**: `scene/light.cpp`（IES 灯光节点）, `kernel/` 下的灯光采样内核

## 关联文件
- `util/math.h` - 数学函数（`fabsf`、`M_PI_F` 等）
- `scene/light.cpp` - 灯光场景对象中加载 IES 数据
