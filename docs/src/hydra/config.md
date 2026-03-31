# config.h - Hydra渲染代理命名空间与前向声明配置

## 概述

本文件是 Cycles Hydra渲染代理 模块的全局配置头文件，定义了命名空间宏和所有关键类的前向声明。它是几乎所有 `hydra/` 目录下源文件的共同基础依赖，确保命名空间一致性和编译依赖最小化。

## 类与结构体

### 前向声明的类（namespace HD_CYCLES_NS / HdCycles）

- `HdCyclesCamera` -- 相机场景图元（Sprim）
- `HdCyclesDelegate` -- 渲染委托核心类
- `HdCyclesSession` -- 渲染会话封装
- `HdCyclesRenderBuffer` -- 渲染缓冲区 Bprim

### 前向声明的类（namespace CCL_NS / ccl）

- `AttributeSet` -- 属性集合
- `BufferParams` -- 缓冲区参数
- `Camera` -- 相机
- `Geometry` -- 几何体基类
- `Hair` -- 毛发几何体
- `Light` -- 灯光
- `Mesh` -- 网格几何体
- `Object` -- 场景对象（实例）
- `ParticleSystem` -- 粒子系统
- `Pass` -- 渲染通道
- `PointCloud` -- 点云几何体
- `Scene` -- 场景
- `Session` -- 渲染会话
- `SessionParams` -- 会话参数
- `Shader` -- 着色器
- `ShaderGraph` -- 着色器图
- `ShaderNode` -- 着色器节点
- `Volume` -- 体积几何体

## 核心函数

无函数定义。

## 依赖关系

- **内部头文件**: 无
- **外部依赖**: `pxr/pxr.h`（通过 IWYU pragma export 导出）
- **被引用**: 几乎所有 `hydra/` 目录下的头文件（20 个文件），是模块的基础依赖

## 实现细节 / 关键算法

1. **命名空间宏定义**:
   - `CCL_NS` -> `ccl`（Cycles 核心命名空间别名）
   - `CCL_NAMESPACE_USING_DIRECTIVE` -> `using namespace ccl;`
   - `HD_CYCLES_NS` -> `HdCycles`（Hydra Cycles 插件命名空间）
   - `HDCYCLES_NAMESPACE_OPEN_SCOPE` -- 打开 `HdCycles` 命名空间，同时引入 `ccl` 和 `pxr` 命名空间
   - `HDCYCLES_NAMESPACE_CLOSE_SCOPE` -- 关闭 `HdCycles` 命名空间

2. **设计意图**: 通过在 `HDCYCLES_NAMESPACE_OPEN_SCOPE` 中同时使用 `CCL_NAMESPACE_USING_DIRECTIVE` 和 `PXR_NAMESPACE_USING_DIRECTIVE`，使得 `HdCycles` 命名空间内的代码可以直接使用 Cycles 和 USD 的类型名称，无需添加命名空间前缀，简化了代码书写。

3. **前向声明策略**: 将所有常用类的前向声明集中在此文件中，避免各文件重复声明，同时减少头文件间的包含依赖。

## 关联文件

- 被所有 `hydra/` 目录下的 `.h` 文件包含
- `pxr/pxr.h` -- Pixar USD 基础头文件
