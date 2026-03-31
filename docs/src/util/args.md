# args.h - 命令行参数解析封装

## 概述
本文件是对 OpenImageIO（OIIO）库 `ArgParse` 参数解析器的薄封装，将其引入 Cycles 命名空间。`ArgParse` 提供了命令行参数的定义、解析和帮助信息生成功能，适用于 Cycles 独立应用程序（如 `cycles` 命令行渲染器）的参数处理。

## 类与结构体

### `ArgParse`（来自 OIIO）
通过 `using OIIO::ArgParse;` 引入，提供：
- 注册命令行选项及其回调
- 自动生成帮助信息
- 类型安全的参数绑定

## 核心函数
本文件无自定义函数，仅通过 `using` 声明引入 OIIO 的 `ArgParse` 类。

## 依赖关系
- **外部依赖**: `<OpenImageIO/argparse.h>`
- **被引用**: `app/cycles_standalone.cpp` 等独立应用入口文件

## 关联文件
- `util/string.h` - 参数字符串处理
- `util/log.h` - 日志输出配合参数解析
