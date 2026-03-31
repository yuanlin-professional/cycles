# make_format.py - 代码格式化工具

## 概述

该脚本实现了 `make format` 命令，使用 `clang-format` 对 Cycles 源代码进行自动格式化。它通过 Git 获取源文件列表，并利用多进程并行处理以提高格式化效率。

## 核心函数

### `source_files_from_git(paths)`

从 Git 仓库中获取源文件列表：

- 使用 `git ls-tree -r HEAD --name-only -z` 命令列出指定路径下的所有受版本控制的文件
- 以空字节 (`\0`) 分隔输出，避免文件名中的特殊字符问题
- 返回解码后的 ASCII 文件路径列表

### `clang_format_file(files)`

对一组文件执行 clang-format：

- 调用硬编码路径的 clang-format 工具：`/home/brecht/dev/lib/linux_centos7_x86_64/llvm/bin/clang-format`
- 使用 `-i` 参数进行就地修改，`-verbose` 输出详细信息
- 注意：路径为开发者环境特定路径，实际使用可能需要调整

### `clang_print_output(output)`

打印 clang-format 的输出结果，将字节流解码为 UTF-8 文本。

### `clang_format(files)`

并行格式化的调度函数：

- 创建多进程池（`multiprocessing.Pool`）
- 根据 CPU 核心数计算分块大小（chunk_size），范围在 1 到 32 之间
- 使用 `apply_async` 异步分发格式化任务
- 等待所有任务完成后返回

### `main()`

主入口函数：

1. 将工作目录切换到项目根目录（脚本位置向上两级）
2. 从 `src` 目录获取所有 Git 追踪的源文件
3. 按扩展名过滤，保留以下类型：`.c`、`.cc`、`.cpp`、`.cxx`、`.h`、`.hh`、`.hpp`、`.hxx`、`.osl`、`.glsl`、`.cu`、`.cl`
4. 排除忽略列表中的文件（如 `src/render/sobol.cpp`，因内容过大不适合 clang-format 处理）
5. 执行并行格式化

## 依赖关系

### Python 标准库

- `multiprocessing` - 多进程并行
- `os` - 文件路径操作
- `sys` - 系统接口
- `subprocess` - 外部命令调用

### 外部工具

- `clang-format` - LLVM 代码格式化工具
- `git` - 用于获取源文件列表
