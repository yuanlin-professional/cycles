# types.h - 开放着色语言(OSL)核心类型定义

## 概述

`types.h` 是 Cycles 渲染器中开放着色语言(OSL)集成层的基础类型定义文件。该文件定义了跨 CPU 和 GPU 平台使用的 `DeviceString` 类型、OSL 闭包树的节点结构体(`OSLClosure`、`OSLClosureMul`、`OSLClosureAdd`、`OSLClosureComponent`)、闭包类型枚举 `OSLClosureType`,以及与 OSL 运行时兼容的 `ShaderGlobals` 结构体。此外还定义了 GPU 端纹理句柄的类型编码宏。

## 类与结构体

### `DeviceString` 类型

根据编译目标平台有三种定义:

| 平台 | 类型 | 说明 |
|------|------|------|
| GPU (`__KERNEL_GPU__`) | `size_t` | 字符串以哈希值表示 |
| CPU + OIIO (OSL >= 1.14) | `ustringhash` | OIIO 字符串哈希类型 |
| CPU + OIIO (OSL < 1.14) | `ustring` | OIIO 唯一字符串类型 |
| 其他 | `const char*` | 原始 C 字符串指针 |

### `OSLClosureType` 枚举
所有 OSL 闭包的类型标识符:

| 枚举值 | 说明 |
|--------|------|
| `OSL_CLOSURE_MUL_ID` (-1) | 乘法节点 |
| `OSL_CLOSURE_ADD_ID` (-2) | 加法节点 |
| `OSL_CLOSURE_NONE_ID` (0) | 空闭包 |
| `OSL_CLOSURE_Diffuse_ID` (1+) | 漫反射闭包(及后续通过 closures_template.h 自动生成的所有闭包类型) |
| `OSL_CLOSURE_LAYER_ID` | 分层闭包(手动添加在枚举末尾) |

### `OSLClosure`
闭包树节点的基类:
```cpp
struct OSLClosure {
    OSLClosureType id;
};
```

### `OSLClosureMul`
乘法闭包节点(权重乘以子闭包):
```cpp
struct ccl_align(8) OSLClosureMul : public OSLClosure {
    packed_float3 weight;           // 颜色权重
    const ccl_private OSLClosure *closure;  // 子闭包指针
};
```

### `OSLClosureAdd`
加法闭包节点(两个子闭包之和):
```cpp
struct ccl_align(8) OSLClosureAdd : public OSLClosure {
    const ccl_private OSLClosure *closureA;  // 左子闭包
    const ccl_private OSLClosure *closureB;  // 右子闭包
};
```

### `OSLClosureComponent`
闭包组件节点(叶子节点,实际的 BSDF/发射等):
```cpp
struct ccl_align(8) OSLClosureComponent : public OSLClosure {
    packed_float3 weight;  // 组件权重
    // 具体闭包数据紧跟在此结构体之后(通过指针偏移访问)
};
```

### `ShaderGlobals`
与 `OSL::ShaderGlobals` 内存布局兼容的 Cycles 版本:

**OSL 兼容部分**(布局必须与 `OSL::ShaderGlobals` 完全一致):

| 成员 | 类型 | 说明 |
|------|------|------|
| `P`, `dPdx`, `dPdy` | `packed_float3` | 着色点位置及屏幕空间偏导数 |
| `dPdz` | `packed_float3` | 位置的 z 方向偏导数 |
| `I`, `dIdx`, `dIdy` | `packed_float3` | 入射方向及偏导数 |
| `N` | `packed_float3` | 着色法线 |
| `Ng` | `packed_float3` | 几何法线 |
| `u`, `dudx`, `dudy` | `float` | U 纹理坐标及偏导数 |
| `v`, `dvdx`, `dvdy` | `float` | V 纹理坐标及偏导数 |
| `dPdu`, `dPdv` | `packed_float3` | 位置对 UV 的偏导数 |
| `time`, `dtime` | `float` | 时间及时间步长 |
| `dPdtime` | `packed_float3` | 位置对时间的偏导数 |
| `Ps`, `dPsdx`, `dPsdy` | `packed_float3` | 纹理空间位置及偏导数 |
| `sd` | `ShaderData*` | OSL 中名为 "render-state" 的不透明指针 |
| `tracedata` | `OSLTraceData*` | trace() 数据缓存 |
| `closure_pool` (GPU) / `kg` (CPU) | 指针 | GPU: 闭包内存池; CPU: 线程内核全局数据 |
| `context` | `void*` | 着色上下文 |
| `shadingStateUniform` | `void*` | 着色状态 |
| `thread_index` | `int` | 线程索引 |
| `shade_index` | `int` | 着色索引(GPU 上编码路径状态) |
| `renderer` | `void*` | 渲染器指针 |
| `object2common`, `shader2common` | `void*` | 变换矩阵指针(实际指向 ShaderData) |
| `Ci` | `OSLClosure*` | 闭包输出 |
| `surfacearea` | `float` | 表面面积 |
| `raytype` | `int` | 光线类型(= path_flag) |
| `flipHandedness` | `int` | 手性翻转标志 |
| `backfacing` | `int` | 背面标志 |

**Cycles 扩展部分**(仅 CPU):

| 成员 | 类型 | 说明 |
|------|------|------|
| `path_state` | `const IntegratorStateCPU*` | 积分器路径状态 |
| `shadow_path_state` | `const IntegratorShadowStateCPU*` | 阴影路径状态 |

### `OSLNoiseOptions` / `OSLTextureOptions`
空结构体,作为 OSL 噪声和纹理选项的占位符。

## 核心函数

### `make_string(const char *str, size_t hash)`
```cpp
ccl_device_inline DeviceString make_string(const char *str, const size_t hash)
```
- **功能**: 创建设备字符串。
  - GPU: 忽略字符串,直接返回哈希值
  - CPU + OIIO: 从字符串创建 `ustring`/`ustringhash`,并断言哈希值匹配
  - 其他: 返回原始字符串指针

## 依赖关系

- **内部头文件**:
  - `<OSL/oslversion.h>` - OSL 版本信息(仅 CPU)
  - `kernel/types.h` - 内核基础类型
  - `util/defines.h` - 工具宏定义
  - `util/types_float3.h` - float3 类型
- **被引用**:
  - `src/kernel/osl/osl.h` - 着色器引擎主入口
  - `src/kernel/osl/services.cpp` - CPU 渲染服务
  - `src/kernel/osl/services.h` - 渲染服务声明
  - `src/kernel/osl/camera.h` - 相机着色器
  - `src/kernel/osl/closures_setup.h` - 闭包设置函数

## 实现细节 / 关键算法

### ShaderGlobals 布局兼容性

`ShaderGlobals` 的设计核心在于与 `OSL::ShaderGlobals` 保持内存布局兼容。代码通过 `static_assert` 验证两个关键约束:
1. `sizeof(ShaderGlobals) >= sizeof(OSL::ShaderGlobals)` - Cycles 版本不小于 OSL 版本
2. `offsetof(ShaderGlobals, backfacing) == offsetof(OSL::ShaderGlobals, backfacing)` - 最后一个共享字段的偏移量一致

Cycles 的 `ShaderGlobals` 在 OSL 结构体末尾扩展了 `path_state` 和 `shadow_path_state` 指针(仅 CPU),这些额外字段不影响 OSL 的正常运作。

### 不透明指针的重用

OSL 的 `ShaderGlobals` 中有几个 "不透明指针" 字段,Cycles 重新定义了它们的用途:
- `render-state` (OSL) -> `sd` (Cycles): 指向 ShaderData
- `objdata` (OSL) -> `closure_pool` (GPU) 或 `kg` (CPU)
- `object2common` / `shader2common`: 均指向 ShaderData,由渲染服务在矩阵查询时使用

### 闭包树结构

OSL 着色器的输出 `Ci` 是一棵闭包树:
- **内部节点**: `OSLClosureMul`(权重乘法)和 `OSLClosureAdd`(加法)
- **叶子节点**: `OSLClosureComponent`,具体闭包参数数据紧跟在结构体后面(柔性数组风格)
- **特殊节点**: `LAYER` 类型,用于分层材质

所有闭包结构体按 8 字节对齐(`ccl_align(8)`)以确保跨平台一致性。

### GPU 纹理句柄宏

```cpp
#define OSL_TEXTURE_HANDLE_TYPE_IES         ((uintptr_t)0x2 << 30)
#define OSL_TEXTURE_HANDLE_TYPE_SVM         ((uintptr_t)0x1 << 30)
#define OSL_TEXTURE_HANDLE_TYPE_AO_OR_BEVEL ((uintptr_t)0x3 << 30)
```

纹理句柄在 GPU 上编码为 32 位整数:高 2 位表示类型,低 30 位表示槽位索引。提取宏:
- `OSL_TEXTURE_HANDLE_TYPE(handle)`: 提取类型位
- `OSL_TEXTURE_HANDLE_SLOT(handle)`: 提取槽位索引

## 关联文件

- `src/kernel/osl/closures_template.h` - 通过 X-Macro 生成 `OSLClosureType` 枚举值
- `src/kernel/osl/closures_setup.h` - 使用本文件的闭包类型定义
- `src/kernel/osl/osl.h` - 使用 `ShaderGlobals` 和 `OSLClosure` 进行着色器求值
- `src/kernel/osl/services_gpu.h` - GPU 端使用 `DeviceString` 和纹理句柄宏
- `src/kernel/osl/compat.h` - 提供 `OSLUStringHash` 等兼容类型
