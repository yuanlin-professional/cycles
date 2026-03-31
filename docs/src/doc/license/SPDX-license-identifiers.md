# SPDX-license-identifiers.txt - SPDX 许可证标识符对照表

## 概述

此文件是 Cycles 项目中使用的 SPDX（Software Package Data Exchange）许可证标识符的对照索引表。SPDX 是一种用于标准化软件许可证标识的国际规范，Cycles 项目在所有源代码文件的头部使用 SPDX 标识符来声明文件的许可证类型。

## 关键条款

### 许可证标识符对照

| SPDX 标识符 | 许可证文件 | 官方链接 |
|-------------|-----------|---------|
| `Apache-2.0` | `Apache-2-license.txt` | https://spdx.org/licenses/Apache-2.0.html |
| `BSD-3-Clause` | `BSD-3-Clause-license.txt` | https://spdx.org/licenses/BSD-3-Clause.html |
| `MIT` | `MIT-license.txt` | https://spdx.org/licenses/MIT.html |
| `Zlib` | `Zlib-license.txt` | https://spdx.org/licenses/Zlib.html |

### 使用方式

在 Cycles 源代码文件中，许可证通过文件头部的 SPDX 注释进行声明，格式如下：

```
# SPDX-FileCopyrightText: 2011-2022 Blender Foundation
#
# SPDX-License-Identifier: Apache-2.0
```

其中 `SPDX-License-Identifier` 后的值对应本表中的标识符，开发者可通过查阅此对照表找到对应的完整许可证文本。

## 在 Cycles 中的作用

此文件为 Cycles 开发者和用户提供了一个快速参考，便于将源代码中的 SPDX 标识符映射到具体的许可证全文。这种做法遵循了现代开源项目的最佳实践，使许可证管理更加规范和高效。
