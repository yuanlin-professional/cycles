# shader_eval.h / shader_eval.cpp - 着色器求值器

## 概述

本文件实现了 Cycles 渲染器的着色器求值类 `ShaderEval`，用于在渲染前预计算特定着色器效果。它支持四种求值类型：置换（Displace）、背景光（Background）、曲线阴影透明度（Curve Shadow Transparency）和体积密度（Volume Density）。该类自动根据设备类型选择 CPU 或 GPU 执行路径。

## 类与结构体

### ShaderEval

- **继承**: 无
- **功能**: 将着色器求值抽象为输入/输出的批量处理：用户提供输入点（`KernelShaderEvalInput`），系统在 CPU 或 GPU 上执行着色器内核，返回计算结果（浮点数组）。
- **关键成员**:
  - `device_` — 执行求值的设备指针
  - `progress_` — 进度管理器引用，用于取消检测
- **关键方法**:
  - `eval()` — 主入口函数，接受求值类型、输入填充回调和输出读取回调
  - `eval_cpu()` — CPU 执行路径，使用 TBB 并行化
  - `eval_gpu()` — GPU 执行路径，使用设备队列分块执行

### ShaderEvalType（枚举）

- `SHADER_EVAL_DISPLACE` — 网格置换求值
- `SHADER_EVAL_BACKGROUND` — 背景/环境光求值
- `SHADER_EVAL_CURVE_SHADOW_TRANSPARENCY` — 毛发曲线阴影透明度求值
- `SHADER_EVAL_VOLUME_DENSITY` — 体积密度求值（用于体积数据烘焙）

## 核心函数

### eval()

统一的着色器求值入口：

1. 遍历设备（目前仅使用第一个设备，多设备尚未完全支持）
2. 分配输入和输出设备向量
3. 调用用户的 `fill_input` 回调填充输入数据
4. 将输入拷贝到设备，输出清零
5. 根据设备类型调用 `eval_cpu()` 或 `eval_gpu()`
6. 成功后从设备拷回输出，调用 `read_output` 回调
7. 释放设备内存

### eval_cpu()

CPU 端着色器求值：

1. 获取 CPU 内核线程全局状态
2. 使用 `tbb::task_arena` 限制线程数（匹配 CPU 设备线程数）
3. 在 `parallel_for` 中逐点调用对应的内核函数：
   - `kernels.shader_eval_displace()`
   - `kernels.shader_eval_background()`
   - `kernels.shader_eval_curve_shadow_transparency()`
   - `kernels.shader_eval_volume_density()`
4. 每个工作项检查取消状态

### eval_gpu()

GPU 端着色器求值：

1. 根据求值类型映射到设备内核枚举
2. 创建 GPU 队列并初始化执行环境
3. 以 65536 为分块大小，逐块提交内核
4. 每块完成后同步并检查取消状态

## 依赖关系

- **内部头文件**:
  - `device/memory.h` — `device_vector` 设备内存
  - `kernel/types.h` — `KernelShaderEvalInput` 输入结构体
  - `device/device.h` — 设备接口
  - `device/queue.h` — GPU 设备队列
  - `device/cpu/kernel.h` — CPU 内核函数
  - `kernel/device/cpu/globals.h` — CPU 内核全局状态
  - `util/progress.h` — 进度管理
  - `util/tbb.h` — TBB 并行工具
- **被引用**:
  - `scene/mesh_displace.cpp` — 网格置换
  - `scene/light.cpp` — 背景光采样
  - `scene/hair.cpp` — 毛发阴影透明度
  - `bvh/octree.cpp` — 体积密度八叉树构建

## 实现细节 / 关键算法

### 函数式回调设计

`eval()` 使用 `std::function` 回调将输入填充和输出读取与求值逻辑解耦：
- `fill_input(device_vector<KernelShaderEvalInput>&)` — 返回实际输入点数
- `read_output(device_vector<float>&)` — 处理输出结果

这种设计允许调用方灵活控制数据格式和存储方式。

### GPU 分块执行

GPU 执行使用固定 65536 的分块大小以支持取消操作。每块执行后同步并检查 `progress_.get_cancel()`，避免长时间无响应。工作尺寸被断言不超过 `0x7fffffff`（int32 上限）。

### CPU 线程控制

CPU 执行使用 `tbb::task_arena` 创建独立的执行上下文，线程数限制为 `device->info.cpu_threads`，避免与其他 TBB 任务竞争。每个线程使用独立的 `ThreadKernelGlobalsCPU` 全局状态。

### 多设备限制

当前实现仅在第一个设备上执行求值，对多设备场景记录调试日志但不做实际处理。

## 关联文件

- `src/scene/mesh_displace.cpp` — 调用者：网格置换计算
- `src/scene/light.cpp` — 调用者：背景光采样
- `src/scene/hair.cpp` — 调用者：毛发阴影透明度
- `src/bvh/octree.cpp` — 调用者：体积密度烘焙
- `src/device/cpu/kernel.h` — CPU 内核函数定义
