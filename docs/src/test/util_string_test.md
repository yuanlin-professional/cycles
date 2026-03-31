# util_string_test.cpp - 字符串工具函数测试

## 概述

此文件测试 Cycles 工具库中的字符串操作函数，包括格式化输出、大小写无关比较、分割、替换、后缀判断、前缀判断、空白去除和商标符号移除等功能。

## 测试用例

### string_printf 测试组 - 格式化字符串
| 测试名 | 功能 |
|--------|------|
| `no_format` | 无格式化参数，直接返回原字符串 |
| `int_number` | 整数格式化 `%d`，验证 `"foo 314 bar"` |
| `float_number_default_precision` | 浮点数默认精度 `%f`，验证 `"foo 3.141500 bar"` |
| `float_number_custom_precision` | 浮点数自定义精度 `%.1f`，验证 `"foo 3.1 bar"` |

### string_iequals 测试组 - 大小写无关比较
| 测试名 | 输入 | 期望结果 | 说明 |
|--------|------|----------|------|
| `empty_a` | `"", "foo"` | `false` | 空字符串与非空字符串不等 |
| `empty_b` | `"foo", ""` | `false` | 非空字符串与空字符串不等 |
| `same_register` | `"foo", "foo"` | `true` | 完全相同 |
| `different_register` | `"XFoo", "XfoO"` | `true` | 大小写不同但内容相同 |

### string_split 测试组 - 字符串分割
| 测试名 | 输入 | 期望结果 |
|--------|------|----------|
| `empty` | `""` | 0 个分词 |
| `only_spaces` | `"   \t\t \t"` | 0 个分词 |
| `single` | `"foo"` | 1 个分词: `["foo"]` |
| `simple` | `"foo a bar b"` | 4 个分词: `["foo", "a", "bar", "b"]` |
| `multiple_spaces` | `" \t foo \ta bar b\t  "` | 4 个分词（忽略多余空白） |

### string_replace 测试组 - 字符串替换
| 测试名 | 原字符串 | 查找 | 替换为 | 期望结果 |
|--------|----------|------|--------|----------|
| `empty_haystack_and_other` | `""` | `"x"` | `""` | `""` |
| `empty_haystack` | `""` | `"x"` | `"y"` | `""` |
| `empty_other` | `"x"` | `"x"` | `""` | `""` |
| `long_haystack_empty_other` | `"a x b xxc"` | `"x"` | `""` | `"a  b c"` |
| `long_haystack` | `"a x b xxc"` | `"x"` | `"FOO"` | `"a FOO b FOOFOOc"` |

### string_endswith 测试组 - 后缀判断（util_ 前缀版）
| 测试名 | 字符串 | 后缀 | 期望结果 |
|--------|--------|------|----------|
| `empty_both` | `""` | `""` | `true` |
| `empty_string` | `""` | `"foo"` | `false` |
| `empty_end` | `"foo"` | `""` | `true` |
| `simple_true` | `"foo bar"` | `"bar"` | `true` |
| `simple_false` | `"foo bar"` | `"foo"` | `false` |

### string_strip 测试组 - 空白去除
| 测试名 | 输入 | 期望结果 |
|--------|------|----------|
| `empty` | `""` | `""` |
| `only_spaces` | `"      "` | `""` |
| `no_spaces` | `"foo bar"` | `"foo bar"` |
| `with_spaces` | `"    foo bar "` | `"foo bar"` |

### string_remove_trademark 测试组 - 商标符号移除
| 测试名 | 输入 | 期望结果 | 说明 |
|--------|------|----------|------|
| `empty` | `""` | `""` | 空字符串 |
| `no_trademark` | `"foo bar"` | `"foo bar"` | 无商标符号 |
| `only_tm` | `"foo bar(TM) zzz"` | `"foo bar zzz"` | 仅移除 (TM) |
| `only_r` | `"foo bar(R) zzz"` | `"foo bar zzz"` | 仅移除 (R) |
| `both` | `"foo bar(TM)(R) zzz"` | `"foo bar zzz"` | 移除 (TM) 和 (R) |
| `both_space` | `"foo bar(TM) (R) zzz"` | `"foo bar zzz"` | 带空格移除 |
| `both_space_around` | `"foo bar (TM) (R) zzz"` | `"foo bar zzz"` | 前后带空格 |
| `trademark_space_suffix` | `"foo bar (TM)"` | `"foo bar"` | 末尾 (TM) |
| `trademark_space_middle` | `"foo bar (TM) baz"` | `"foo bar baz"` | 中间 (TM) |
| `r_space_suffix` | `"foo bar (R)"` | `"foo bar"` | 末尾 (R) |
| `r_space_middle` | `"foo bar (R) baz"` | `"foo bar baz"` | 中间 (R) |

### string_startswith 测试组 - 前缀判断
- **功能**: 验证 `string_startswith` 函数在各种边界条件下的正确性，包括空字符串、完全匹配、部分匹配和不匹配的情况。

### string_endswith 测试组 - 后缀判断（基础版）
- **功能**: 验证 `string_endswith` 函数在各种边界条件下的正确性，包括空字符串、完全匹配、部分匹配和不匹配的情况。

## 依赖关系
- **被测源文件**: `util/string.h`
- **测试框架**: Google Test (GTest)

## 关联文件
- `src/util/string.h` - 字符串工具函数声明
- `src/util/string.cpp` - 字符串工具函数实现
