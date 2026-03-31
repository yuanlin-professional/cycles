# make_utils.py - 构建工具通用函数库

## 概述

该文件是 `make_update.py` 等构建脚本的基础工具库，提供了命令执行、Git 操作（分支检测、子模块管理、配置读写）以及命令可用性检测等通用函数。它是 Cycles 构建自动化流程的底层支撑模块。

## 核心函数

### 命令执行

#### `call(cmd, exit_on_error=True, silent=False, env=None)`

执行外部命令的通用包装器：

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `cmd` | `Sequence[str]` | - | 要执行的命令及参数 |
| `exit_on_error` | `bool` | `True` | 命令失败时是否退出程序 |
| `silent` | `bool` | `False` | 是否抑制输出（stdout 和 stderr 重定向到 DEVNULL） |
| `env` | `Optional[Dict[str, str]]` | `None` | 额外的环境变量（合并到当前环境中） |

特点：
- 非静默模式下，打印完整命令行（包含环境变量）
- 在 Windows 上显式刷新 stdout/stderr 以确保输出顺序正确
- 返回命令退出码

#### `check_output(cmd, exit_on_error=True)`

执行命令并捕获输出：

- 将 stderr 合并到 stdout
- 使用 `universal_newlines=True` 确保文本模式输出
- 返回去除首尾空白的字符串
- 异常处理：失败时根据 `exit_on_error` 参数决定退出或返回空串

### Git 分支操作

#### `git_local_branch_exists(git_command, branch)`

检查本地分支是否存在，使用 `git rev-parse --verify` 验证。

#### `git_remote_branch_exists(git_command, remote, branch)`

检查远程分支是否存在，验证 `remotes/{remote}/{branch}` 引用。

#### `git_branch_exists(git_command, branch)`

综合分支检查：依次检查本地分支、`upstream` 远程分支、`origin` 远程分支。

#### `git_branch(git_command)`

获取当前分支名称，使用 `git rev-parse --abbrev-ref HEAD`。

### Git 远程操作

#### `git_get_remote_url(git_command, remote_name)`

获取远程仓库的 URL。

#### `git_remote_exist(git_command, remote_name)`

检查指定名称的远程仓库是否已配置。利用 `git ls-remote --get-url` 的行为：如果远程存在返回 URL，不存在则原样返回名称。

#### `git_is_remote_repository(git_command, repo)`

验证给定仓库地址是否为有效的、可克隆的 Git 仓库。

### Git 配置操作

#### `git_get_config(git_command, key, file=None)`

读取 Git 配置值，支持指定配置文件。

#### `git_set_config(git_command, key, value, file=None)`

设置 Git 配置值，支持指定配置文件。

### Git 子模块管理

#### `_git_submodule_config_key(submodule_dir, key)`

内部函数，生成子模块配置键名，格式为 `submodule.{dir}.{key}`（使用 POSIX 路径格式）。

#### `is_git_submodule_enabled(git_command, submodule_dir)`

检查子模块是否已启用：

- 通过 `.gitmodules` 文件验证子模块存在性
- 检查本地配置中的 `update` 策略
- 如果 update 策略为 `none` 则视为禁用
- 未配置 update 策略时默认为 `checkout`（视为启用）

#### `git_enable_submodule(git_command, submodule_dir)`

启用子模块，将其 `update` 策略设置为 `checkout`。

#### `git_update_submodule(git_command, submodule_dir)`

更新子模块，采用两阶段更新策略：

1. **第一阶段**：设置 `GIT_LFS_SKIP_SMUDGE=1`，执行 `git submodule update --init --progress`，检出目标提交但跳过 LFS 文件下载
2. **第二阶段**：执行 `git lfs pull` 下载 LFS 文件

这种策略的优势：
- 显示下载进度
- 支持断点续传
- 绕过 Git 2.33 以后子模块更新时 LFS 进度被抑制的问题

### 工具函数

#### `command_missing(command)`

检查外部命令是否可用：

- Python 3+：使用 `shutil.which()` 检测
- Python 2：总是返回 `False`（兼容 macOS 旧版 Python）

## 依赖关系

### Python 标准库

- `re` - 正则表达式
- `os` - 文件系统操作
- `shutil` - 命令查找（`which`）
- `subprocess` - 子进程管理
- `sys` - 系统接口
- `pathlib.Path` - 路径对象
- `typing` - 类型注解（`Dict`、`Sequence`、`Optional`）
