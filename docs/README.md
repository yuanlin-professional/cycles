# Cycles 渲染器源码文档

## 概述

Cycles 是 Blender 的物理级路径追踪渲染器，同时支持作为独立渲染器使用。它采用单向路径追踪算法，支持 CPU 和多种 GPU 后端（CUDA、OptiX、HIP、Metal、oneAPI），能够生成照片级真实感渲染图像。

本文档为 Cycles 源码的中文技术文档，按模块组织，覆盖所有源码目录，旨在帮助开发者理解 Cycles 的架构设计和实现细节。

## 整体架构

Cycles 采用分层架构，从底向上依次为：

```
┌──────────────────────────────────────────────────────────┐
│                    应用层 (Application)                    │
│    app/（独立渲染器）   hydra/（USD 渲染代理）   test/     │
├──────────────────────────────────────────────────────────┤
│                   集成层 (Integration)                     │
│         integrator/（渲染调度）  session/（会话管理）       │
├──────────────────────────────────────────────────────────┤
│                    场景层 (Scene)                          │
│    scene/（场景描述：几何体、着色器、光源、相机、胶片）      │
├──────────────────────────────────────────────────────────┤
│                    内核层 (Kernel)                         │
│  kernel/（路径追踪内核、着色器虚拟机、闭包、几何求交）      │
├──────────────────────────────────────────────────────────┤
│                 设备抽象层 (Device)                        │
│  device/（CPU、CUDA、OptiX、HIP、Metal、oneAPI 后端）     │
├──────────────────────────────────────────────────────────┤
│                  加速结构 (BVH)                            │
│         bvh/（层次包围体构建与遍历）                       │
├──────────────────────────────────────────────────────────┤
│                核心基础设施 (Core)                         │
│        graph/（节点图系统）    subd/（细分曲面）            │
├──────────────────────────────────────────────────────────┤
│                  基础层 (Foundation)                       │
│   util/（数学、类型、线程、内存）  cmake/（构建）  doc/    │
└──────────────────────────────────────────────────────────┘
```

## 渲染流程

1. **场景构建**: `scene/` 模块从 Blender（或 USD/Hydra）接收场景数据，构建内部表示
2. **场景同步**: 将场景数据同步到设备内存（通过 `device/`）
3. **BVH 构建**: `bvh/` 模块为几何体构建加速结构
4. **渲染调度**: `integrator/` 模块通过 `session/` 发起渲染
5. **路径追踪**: `kernel/` 中的路径追踪内核在 CPU/GPU 上执行
6. **自适应采样**: 根据像素收敛情况动态分配采样数
7. **降噪**: 可选的 AI 降噪后处理
8. **输出**: 渲染结果写入胶片缓冲区

## 模块依赖关系

```
app, hydra, test
    │
    ▼
integrator ◄──► session
    │               │
    ▼               ▼
  scene ◄────── device
    │               │
    ▼               ▼
   bvh          kernel/
    │           ┌───┴───────────────────────┐
    ▼           ▼       ▼        ▼          ▼
  graph    integrator  svm    closure      geom
  subd     camera     light    film       bvh(k)
    │       sample     osl    device/*
    ▼
  util ◄─── (所有模块共同依赖)
```

## 目录索引

### 基础层

| 目录 | 说明 | 文档链接 |
|------|------|----------|
| `src/util/` | 基础工具库（数学、类型、线程、内存、字符串等） | [util/README.md](src/util/README.md) |
| `src/cmake/` | CMake 构建系统、Find 模块、平台检测 | [cmake/README.md](src/cmake/README.md) |
| `src/doc/` | 许可证信息、预计算工具 | [doc/README.md](src/doc/README.md) |

### 核心基础设施

| 目录 | 说明 | 文档链接 |
|------|------|----------|
| `src/graph/` | 节点图系统（着色器节点、参数管理） | [graph/README.md](src/graph/README.md) |
| `src/subd/` | 细分曲面（Catmull-Clark 细分、Patch 生成） | [subd/README.md](src/subd/README.md) |

### 加速结构

| 目录 | 说明 | 文档链接 |
|------|------|----------|
| `src/bvh/` | 层次包围体 (BVH) 构建与管理 | [bvh/README.md](src/bvh/README.md) |

### 设备抽象层

| 目录 | 说明 | 文档链接 |
|------|------|----------|
| `src/device/` | 设备抽象层（基类与管理） | [device/README.md](src/device/README.md) |
| `src/device/cpu/` | CPU 设备后端 | [device/cpu/README.md](src/device/cpu/README.md) |
| `src/device/cuda/` | CUDA 设备后端（NVIDIA GPU） | [device/cuda/README.md](src/device/cuda/README.md) |
| `src/device/hip/` | HIP 设备后端（AMD GPU） | [device/hip/README.md](src/device/hip/README.md) |
| `src/device/hiprt/` | HIPRT 设备后端（AMD 光线追踪） | [device/hiprt/README.md](src/device/hiprt/README.md) |
| `src/device/optix/` | OptiX 设备后端（NVIDIA RTX 光追） | [device/optix/README.md](src/device/optix/README.md) |
| `src/device/metal/` | Metal 设备后端（Apple GPU） | [device/metal/README.md](src/device/metal/README.md) |
| `src/device/oneapi/` | oneAPI 设备后端（Intel GPU） | [device/oneapi/README.md](src/device/oneapi/README.md) |
| `src/device/multi/` | 多设备后端（多 GPU 协同） | [device/multi/README.md](src/device/multi/README.md) |
| `src/device/dummy/` | 虚拟设备后端（测试用） | [device/dummy/README.md](src/device/dummy/README.md) |

### 内核层

| 目录 | 说明 | 文档链接 |
|------|------|----------|
| `src/kernel/` | 渲染内核根目录（类型定义与入口） | [kernel/README.md](src/kernel/README.md) |
| `src/kernel/integrator/` | 路径追踪积分器内核 | [kernel/integrator/README.md](src/kernel/integrator/README.md) |
| `src/kernel/svm/` | 着色器虚拟机 (SVM) 节点实现 | [kernel/svm/README.md](src/kernel/svm/README.md) |
| `src/kernel/closure/` | BSDF 闭包实现 | [kernel/closure/README.md](src/kernel/closure/README.md) |
| `src/kernel/geom/` | 几何体求交内核 | [kernel/geom/README.md](src/kernel/geom/README.md) |
| `src/kernel/light/` | 光源采样内核 | [kernel/light/README.md](src/kernel/light/README.md) |
| `src/kernel/film/` | 胶片/成像平面内核 | [kernel/film/README.md](src/kernel/film/README.md) |
| `src/kernel/camera/` | 相机光线生成内核 | [kernel/camera/README.md](src/kernel/camera/README.md) |
| `src/kernel/bvh/` | BVH 遍历内核 | [kernel/bvh/README.md](src/kernel/bvh/README.md) |
| `src/kernel/sample/` | 采样模式生成内核 | [kernel/sample/README.md](src/kernel/sample/README.md) |
| `src/kernel/bake/` | 烘焙内核 | [kernel/bake/README.md](src/kernel/bake/README.md) |
| `src/kernel/util/` | 内核工具函数 | [kernel/util/README.md](src/kernel/util/README.md) |
| `src/kernel/osl/` | 开放着色语言 (OSL) 集成 | [kernel/osl/README.md](src/kernel/osl/README.md) |
| `src/kernel/osl/shaders/` | OSL 着色器节点库（113 个着色器） | [kernel/osl/shaders/README.md](src/kernel/osl/shaders/README.md) |

### 内核设备编译入口

| 目录 | 说明 | 文档链接 |
|------|------|----------|
| `src/kernel/device/gpu/` | GPU 通用内核入口 | [kernel/device/gpu/README.md](src/kernel/device/gpu/README.md) |
| `src/kernel/device/cpu/` | CPU 内核入口 | [kernel/device/cpu/README.md](src/kernel/device/cpu/README.md) |
| `src/kernel/device/cuda/` | CUDA 内核编译入口 | [kernel/device/cuda/README.md](src/kernel/device/cuda/README.md) |
| `src/kernel/device/hip/` | HIP 内核编译入口 | [kernel/device/hip/README.md](src/kernel/device/hip/README.md) |
| `src/kernel/device/hiprt/` | HIPRT 内核编译入口 | [kernel/device/hiprt/README.md](src/kernel/device/hiprt/README.md) |
| `src/kernel/device/optix/` | OptiX 内核编译入口 | [kernel/device/optix/README.md](src/kernel/device/optix/README.md) |
| `src/kernel/device/metal/` | Metal 内核编译入口 | [kernel/device/metal/README.md](src/kernel/device/metal/README.md) |
| `src/kernel/device/oneapi/` | oneAPI 内核编译入口 | [kernel/device/oneapi/README.md](src/kernel/device/oneapi/README.md) |

### 场景层

| 目录 | 说明 | 文档链接 |
|------|------|----------|
| `src/scene/` | 场景描述层（几何体、着色器、光源、相机、胶片） | [scene/README.md](src/scene/README.md) |

### 集成层

| 目录 | 说明 | 文档链接 |
|------|------|----------|
| `src/integrator/` | 积分器（渲染调度、自适应采样、降噪） | [integrator/README.md](src/integrator/README.md) |
| `src/session/` | 渲染会话管理 | [session/README.md](src/session/README.md) |

### 应用层

| 目录 | 说明 | 文档链接 |
|------|------|----------|
| `src/hydra/` | USD/Hydra 渲染代理 | [hydra/README.md](src/hydra/README.md) |
| `src/app/` | 独立应用程序（命令行渲染器） | [app/README.md](src/app/README.md) |
| `src/app/opengl/` | OpenGL 显示后端 | [app/opengl/README.md](src/app/opengl/README.md) |
| `src/test/` | 单元测试 | [test/README.md](src/test/README.md) |

## 外部依赖

| 库名称 | 用途 | 必需/可选 |
|--------|------|-----------|
| OpenImageIO (OIIO) | 图像文件 I/O | 必需 |
| OpenColorIO (OCIO) | 颜色管理 | 可选 |
| OpenShadingLanguage (OSL) | CPU 着色器编译 | 可选 |
| Embree | CPU 光线追踪加速 | 可选 |
| TBB | 任务并行 | 必需 |
| OpenEXR / Imath | HDR 图像 | 必需 |
| CUDA Toolkit | NVIDIA GPU 支持 | 可选 |
| OptiX SDK | NVIDIA RTX 光追 | 可选 |
| HIP SDK | AMD GPU 支持 | 可选 |
| Metal | Apple GPU 支持 | 可选 (macOS) |
| oneAPI (DPC++) | Intel GPU 支持 | 可选 |
| OpenVDB / NanoVDB | 体积数据 | 可选 |
| Intel OIDN | AI 降噪 | 可选 |
| Intel OPG | 路径引导 | 可选 |
| USD (Pixar) | Hydra 渲染代理 | 可选 |
| Alembic | 几何缓存 | 可选 |
| GLEW / OpenGL | 视口显示 | 可选 |

## 术语表

| 英文 | 中文 | 说明 |
|------|------|------|
| Path tracing | 路径追踪 | Cycles 核心渲染算法 |
| BVH | 层次包围体 (BVH) | 光线求交加速结构 |
| Shader | 着色器 | 材质/表面属性描述 |
| Closure / BSDF | 闭包 / 双向散射分布函数 | 描述光线与表面的交互 |
| Kernel | 内核 | GPU/CPU 上执行的渲染计算单元 |
| Device | 设备 | CPU 或 GPU 计算后端 |
| Scene | 场景 | 渲染场景的高层描述 |
| Integrator | 积分器 | 路径追踪渲染调度器 |
| Denoiser | 降噪器 | AI 降噪后处理 |
| SVM | 着色器虚拟机 (SVM) | GPU 兼容的着色器执行引擎 |
| OSL | 开放着色语言 (OSL) | CPU 端着色器编程语言 |
| Wavefront | 波前 | GPU 路径追踪的并行调度方式 |
| Film | 胶片/成像平面 | 渲染结果的像素缓冲区 |
| Light tree | 光源树 | 多光源高效采样的树结构 |
| Adaptive sampling | 自适应采样 | 按收敛度动态分配采样数 |

## 构建方式

```bash
# 独立构建
mkdir build && cd build
cmake ../cycles
make -j$(nproc)

# 作为 Blender 插件构建
# 由 Blender 的构建系统自动处理
```

## 许可证

Cycles 采用 Apache License 2.0 许可证。
