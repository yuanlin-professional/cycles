# compat.h - 开放着色语言(OSL)版本兼容性适配层

## 概述

`compat.h` 是一个轻量级的版本兼容层,用于处理不同版本的开放着色语言(OSL)库之间的 API 差异。该文件主要定义了 `OSLUStringHash` 和 `OSLUStringRep` 类型别名,并提供了哈希值到 `ustring` 的转换函数,确保 Cycles 能够在 OSL 1.14+ 及更早版本上正确编译和运行。

## 类与结构体

本文件无独立类或结构体,仅定义类型别名:

### 类型别名

- **`OSLUStringHash`**: 始终别名为 `OSL::ustringhash`,用于字符串哈希查找。
- **`OSLUStringRep`**: 字符串的内部表示类型。
  - OSL >= 1.14: 别名为 `OSL::ustringhash`(基于哈希的表示)
  - OSL < 1.14: 别名为 `OSL::ustringrep`(旧式表示)

## 核心函数

### `to_ustring(OSLUStringHash h)`
```cpp
static inline OSL::ustring to_ustring(OSLUStringHash h)
```
- **功能**: 将开放着色语言(OSL)字符串哈希值反向查找为对应的 `ustring` 对象。内部调用 `OSL::ustring::from_hash(h.hash())` 实现。
- **用途**: 在需要从哈希值恢复完整字符串时使用,例如调试输出或属性查询。

## 依赖关系

- **内部头文件**:
  - `<OSL/oslconfig.h>` - OSL 配置头文件,提供版本宏和基础类型
- **被引用**:
  - `src/kernel/osl/globals.h` - OSL 全局变量定义中使用 `OSLUStringHash`
  - `src/kernel/osl/services.h` - 渲染服务接口中使用 `OSLUStringHash` 和 `OSLUStringRep`

## 实现细节 / 关键算法

该文件的核心在于版本条件编译。OSL 1.14 引入了 `ustringhash` 作为字符串的统一表示,取代了之前的 `ustringrep`。通过检查 `OSL_LIBRARY_VERSION_CODE` 宏值(>= 11400 表示 1.14+),自动选择正确的类型别名。这种设计使得 Cycles 的其他代码可以统一使用 `OSLUStringHash` 和 `OSLUStringRep`,而无需在每处都进行版本判断。

## 关联文件

- `src/kernel/osl/globals.h` - 使用 `OSLUStringHash` 定义对象名称映射
- `src/kernel/osl/services.h` - 渲染服务函数参数中广泛使用这些类型
- `src/kernel/osl/types.h` - 提供 `DeviceString` 类型,在 GPU 上以哈希值替代字符串
