# md5.h / md5.cpp - MD5 哈希算法实现

## 概述

`md5.h` 和 `md5.cpp` 实现了标准的 MD5 消息摘要算法，主要用于 Cycles 的磁盘缓存系统中生成文件和数据的唯一标识。代码基于 L. Peter Deutsch 的公开实现改编，做了代码精简和风格调整。MD5 生成 128 位（16 字节）的哈希摘要，输出为 32 字符的十六进制字符串。

## 类与结构体

### `class MD5Hash`

MD5 哈希计算类，支持增量式数据追加和最终摘要提取。

**成员变量（protected）：**
- `uint32_t count[2]` -- 消息长度（以位为单位），低位优先（LSW first）
- `uint32_t abcd[4]` -- 摘要缓冲区，存储中间状态和最终结果
- `uint8_t buf[64]` -- 累积块缓冲区，用于处理不足 64 字节的数据片段

**公有方法：**
- `MD5Hash()` -- 构造函数，初始化 MD5 标准初始值（`0x67452301`, `0xefcdab89`, `0x98badcfe`, `0x10325476`）
- `~MD5Hash()` -- 析构函数（默认实现）
- `void append(const uint8_t *data, const int nbytes)` -- 追加原始字节数据到哈希计算中
- `void append(const string &str)` -- 追加字符串数据
- `bool append_file(const string &filepath)` -- 读取文件内容并追加到哈希计算中，以 1024 字节为缓冲区逐块读取；失败时记录错误日志并返回 false
- `string get_hex()` -- 完成计算并返回 32 字符的大写十六进制摘要字符串

**保护方法：**
- `void process(const uint8_t *data)` -- 处理单个 64 字节块，执行 MD5 的四轮变换（F/G/H/I 函数，每轮 16 步，共 64 步）
- `void finish(uint8_t digest[16])` -- 填充消息并输出最终 16 字节摘要

## 核心函数

### `string util_md5_string(const string &str)`

便捷函数，对输入字符串一步计算 MD5 哈希并返回十六进制字符串。内部创建 `MD5Hash` 对象，追加数据后调用 `get_hex()`。

### 实现细节

**`process()` 函数的四轮变换：**
- **Round 1** -- 使用 F(x,y,z) = (x & y) | (~x & z) 函数
- **Round 2** -- 使用 G(x,y,z) = (x & z) | (y & ~z) 函数
- **Round 3** -- 使用 H(x,y,z) = x ^ y ^ z 函数
- **Round 4** -- 使用 I(x,y,z) = y ^ (x | ~z) 函数

每轮使用预定义的 T1-T64 常数（MD5 标准正弦表导出），通过宏 `SET` 执行加法、旋转和混合操作。

**字节序处理：**
- 动态检测大端/小端序
- 小端机器上对齐数据可直接使用，无需拷贝
- 大端机器上需要手动重排字节顺序

## 依赖关系

- **内部头文件**: `util/string.h`（头文件）, `util/log.h`, `util/path.h`（cpp 文件）
- **标准库**: `<cstdio>`, `<cstring>`（cpp 文件）
- **被引用**: `util/path.cpp`, `test/util_md5_test.cpp`, `scene/shader_graph.cpp`, `graph/node.cpp`

## 关联文件

- `util/murmurhash.h` -- 另一种哈希算法（MurmurHash3），用于 Cryptomatte
- `util/hash.h` -- GPU/CPU 通用哈希函数集合
- `util/path.cpp` -- 使用 MD5 生成缓存文件路径
- `scene/shader_graph.cpp` -- 使用 MD5 计算着色器图的唯一标识
- `graph/node.cpp` -- 使用 MD5 计算节点属性的哈希值
