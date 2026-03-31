# murmurhash.h / murmurhash.cpp - MurmurHash3 哈希算法实现

## 概述

`murmurhash.h` 和 `murmurhash.cpp` 实现了 MurmurHash3 的 32 位变体，主要用于 Cryptomatte 渲染通道中对象/材质名称的哈希计算。该算法由 Austin Appleby 编写并置于公共领域，Cycles 从 alShaders/Cryptomatte 项目中引入并做了适配。此外还提供了将哈希值转换为符合 Cryptomatte 规范的浮点数的工具函数。

## 类与结构体

该文件不定义类或结构体。

## 核心函数

### 头文件声明 (murmurhash.h)

- `uint32_t util_murmur_hash3(const void *key, const int len, const uint32_t seed)` -- MurmurHash3 哈希函数，对任意长度的字节数据计算 32 位哈希值
- `float util_hash_to_float(const uint32_t hash)` -- 将 32 位哈希值转换为浮点数，按照 Cryptomatte 规范 1.0 实现

### 实现细节 (murmurhash.cpp)

**内部辅助函数：**
- `uint32_t mm_hash_getblock32(const uint32_t *p, const int i)` -- 块读取，处理字节序和对齐
- `uint32_t mm_hash_fmix32(uint32_t h)` -- 终混操作（avalanche），强制所有位参与最终哈希值的计算

**主哈希函数 `util_murmur_hash3`：**
1. 将输入数据按 4 字节分块处理
2. 每个块经过 c1/c2 常数乘法和旋转混合
3. 尾部不足 4 字节的数据单独处理（switch-case 处理 1/2/3 字节尾部）
4. 最终通过 `mm_hash_fmix32` 做雪崩混合

**哈希转浮点 `util_hash_to_float`：**
- 从 32 位哈希中提取尾数（23 位）、指数（8 位，钳位到 `[1, 254]`）、符号位
- 拼接为 IEEE 754 浮点数的位模式并通过 `memcpy` 转换为 `float`

**平台适配宏：**
- `ROTL32(x, y)` -- MSVC 下使用 `_rotl` 内置函数，其他平台手动实现 32 位左旋

## 依赖关系

- **内部头文件**: `util/math.h`（cpp 文件）, `util/murmurhash.h`（cpp 文件）
- **标准库**: `<cstdint>`（头文件）, `<cstdlib>`, `<cstring>`（cpp 文件）
- **被引用**: `scene/shader.cpp`, `scene/object.cpp`

## 关联文件

- `util/hash.h` -- Cycles 的主哈希函数库（Jenkins Lookup3、PCG 等）
- `util/md5.h` -- 另一种哈希算法实现，用于磁盘缓存
- `scene/shader.cpp` -- 使用 MurmurHash3 计算着色器的 Cryptomatte ID
- `scene/object.cpp` -- 使用 MurmurHash3 计算对象的 Cryptomatte ID
