# util_ies_test.cpp - IES 光域文件解析测试

## 概述

此文件测试 Cycles 工具库中 IES（Illuminating Engineering Society）光域文件解析器的基本功能。IES 文件用于描述灯具的光强分布，广泛用于建筑可视化和照明设计领域的渲染。

## 测试用例

### TEST(util_ies, invalid)
- **功能**: 验证 IES 文件解析器对无效输入的处理。向 `IESFile::load()` 传入一个不符合 IES 格式的字符串 `"Hello, World!"`，期望返回 `false` 表示解析失败。

## 依赖关系
- **被测源文件**: `util/ies.h`
- **测试框架**: Google Test (GTest)

## 关联文件
- `src/util/ies.h` - IESFile 类声明
- `src/util/ies.cpp` - IESFile 类实现
