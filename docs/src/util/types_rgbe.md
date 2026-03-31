# types_rgbe.h - RGBE 高动态范围颜色编码格式

## 概述

`types_rgbe.h` 定义了 RGBE 编码结构体及其与 `float3` RGB 颜色之间的转换函数。RGBE 格式将 RGB 颜色压缩为 4 字节：每个通道 8 位尾数、3 位符号位和 5 位共享指数。该实现基于 GL_EXT_texture_shared_exponent 规范改进，支持负值表示，可表示的最大值约为 65280，最小正值约为 1.19e-7。该格式用于胶片写入等需要高动态范围但内存受限的场景。

## 类与结构体

### `struct RGBE`

4 字节紧凑的 HDR 颜色编码：

```
内存布局: xxxxxxxx xxxxxxxx xxxxxxxx xxx xxxxx
            m(R)      m(G)      m(B)   sgn  exp
```

| 成员 | 类型 | 说明 |
|------|------|------|
| `r` | `uint8_t` | 红色通道 8 位尾数 |
| `g` | `uint8_t` | 绿色通道 8 位尾数 |
| `b` | `uint8_t` | 蓝色通道 8 位尾数 |
| `e` | `uint8_t` | 5 位共享指数 + 3 位符号位 |
| `f` | `float` | 与以上成员共享内存的浮点解释（联合体） |

**关键常量**：
- `RGBE_EXP_BIAS = 15` — 指数偏移量
- `RGBE_MANTISSA_BITS = 8` — 尾数位数
- `RGBE_EXPONENT_BITS = 5` — 指数位数
- `RGBE_MAX = 65280.0f` — 可表示的最大值

通过 `static_assert` 确保结构体大小恰好为 4 字节。

## 核心函数/运算符重载

| 函数签名 | 功能说明 |
|----------|----------|
| `RGBE rgb_to_rgbe(float3 rgb)` | 将 `float3` RGB 颜色编码为 RGBE 格式。使用 `round` 代替传统的 `floor` 以减少累积误差，处理溢出和符号位提取 |
| `float3 rgbe_to_rgb(RGBE rgbe)` | 将 RGBE 解码为 `float3` RGB 颜色。提取共享指数，重建浮点值并恢复符号位 |

### 编码公式

每个颜色分量的值表示为：
```
f = (-1)^sgn * 0.m * 2^(exp - bias)
```

## 依赖关系

- **内部头文件**:
  - `util/math_fast.h` — 快速数学函数（`floor_log2f`、`power_of_2` 等）
  - `util/math_float3.h` — `float3` 数学运算（`fabs`、`reduce_max`、`round` 等）
  - `util/types_base.h`
- **被引用**: `util/types.h`、`kernel/film/write.h`、`test/util_rgbe_test.cpp`

## 关联文件

- `src/kernel/film/write.h` — 胶片写入时使用 RGBE 编码
- `src/test/util_rgbe_test.cpp` — RGBE 编码/解码的单元测试
- `src/util/math_fast.h` — 快速数学工具函数
