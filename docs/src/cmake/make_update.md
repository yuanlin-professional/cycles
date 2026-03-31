# make_update.py - 仓库与预编译库更新工具

## 概述

该脚本实现了 `make update` 命令，用于更新 Cycles Git 仓库和对应平台的预编译库子模块。对于发布分支，它会检出相应的库分支。脚本支持多平台（Linux、macOS、Windows）和多架构（x64、arm64），并提供跳过更新、使用旧版库等选项。

## 核心函数

### `print_stage(text)`

打印阶段性提示信息，在文本前后各添加空行，用于在控制台中清晰分隔不同操作阶段。

### `parse_arguments()`

解析命令行参数：

| 参数 | 类型 | 说明 |
|------|------|------|
| `--no-libraries` | 布尔标志 | 跳过预编译库更新 |
| `--no-cycles` | 布尔标志 | 跳过 Cycles 仓库更新 |
| `--legacy` | 布尔标志 | 使用旧版兼容库 |
| `--git-command` | 字符串 | 自定义 Git 命令路径（默认 `git`） |
| `--architecture` | 字符串 | 手动指定目标架构（`x86_64`、`amd64` 或 `arm64`） |

### `get_effective_platform(args)`

检测当前运行平台：

- `darwin` 映射为 `macos`
- `win32` 映射为 `windows`
- 其他直接使用 `sys.platform`
- 断言确保平台为 `linux`、`macos` 或 `windows` 之一

### `get_effective_architecture(args)`

检测或确定目标 CPU 架构：

- 如果通过 `--architecture` 参数指定，直接使用指定值
- 否则检查 `platform.version()` 是否包含 `ARM64`（用于在 x86_64 Python 二进制下检测 ARM64 主机）
- 回退到 `platform.machine().lower()`
- 将 `x86_64` 和 `amd64` 统一归一化为 `x64`
- 断言最终结果为 `x64` 或 `arm64`

### `ensure_git_lfs(args)`

确保 Git LFS 已安装：

- 使用 `--skip-repo` 参数避免在当前仓库创建 Git 钩子
- 失败时退出程序

### `libraries_update(args)`

更新预编译库子模块：

1. 检测平台和架构
2. 根据 `--legacy` 标志选择正常库路径（`lib/{platform}_{arch}`）或旧版库路径（`lib/legacy/{platform}_{arch}`）
3. 调用 `make_utils.git_enable_submodule` 启用子模块
4. 调用 `make_utils.git_update_submodule` 更新子模块

### `git_update_skip(args, check_remote_exists=True)`

检测仓库是否可以安全更新，返回跳过原因字符串（空串表示可以更新）：

检测项：
- Git 命令是否可用
- 是否正在进行 rebase 或 merge 操作
- 是否有未提交的更改
- 是否配置了远程上游分支

### `cycles_update(args)`

执行 Cycles 仓库更新：

- 使用 `git pull --rebase` 拉取并变基最新代码

### 主程序流程

1. 解析命令行参数
2. 如果未跳过 Cycles 更新：检查仓库状态，安全时执行更新
3. 如果未跳过库更新：更新预编译库子模块
4. 在结尾报告所有被跳过的仓库（确保用户不会遗漏警告信息）

## 依赖关系

### Python 标准库

- `argparse` - 命令行参数解析
- `os` - 文件系统操作
- `platform` - 平台信息检测
- `shutil` - 文件工具
- `sys` - 系统接口
- `pathlib.Path` - 路径对象

### 内部模块

- `make_utils` - 提供 `call`、`check_output`、`command_missing`、`git_enable_submodule`、`git_update_submodule` 等工具函数
