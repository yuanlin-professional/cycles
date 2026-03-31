# util_md5_test.cpp - MD5 哈希计算测试

## 概述

此文件测试 Cycles 工具库中的 MD5 哈希计算函数。MD5 在 Cycles 中主要用于纹理缓存键生成、场景变更检测等需要快速生成内容摘要的场景。

## 测试用例

### TEST(util, util_md5_string)
- **功能**: 验证 `util_md5_string` 函数对给定字符串计算 MD5 哈希值的正确性。
- **验证数据**: 输入 `"Hello, World!"`，期望输出大写十六进制哈希值 `"65A8E27D8879283831B664BD8B7F0AD4"`
- **参考验证方法**: 该期望值通过命令行 `echo -n "Hello, World!" | md5 | tr '[:lower:]' '[:upper:]'` 独立计算得出

## 依赖关系
- **被测源文件**: `util/md5.h`
- **测试框架**: Google Test (GTest)

## 关联文件
- `src/util/md5.h` - MD5 计算函数声明
- `src/util/md5.cpp` - MD5 计算函数实现
