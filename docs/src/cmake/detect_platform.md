# detect_platform.cmake - 平台检测与 macOS 部署目标设置

## 概述

该文件负责在构建配置的早期阶段检测目标平台信息。目前仅针对 macOS (Apple) 平台进行检测和配置，设置 CPU 架构和最低部署目标版本。该文件应在其他构建配置文件之前被包含，以确保平台变量在后续使用时已正确设置。

## 构建选项 / 变量

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `CMAKE_OSX_ARCHITECTURES` | 自动检测（通过 `uname -m`） | macOS 目标 CPU 架构（如 `x86_64` 或 `arm64`） |
| `CMAKE_OSX_DEPLOYMENT_TARGET` | `11.2` | macOS 最低部署目标版本 |

## 关键逻辑

### macOS 架构检测

仅在 Apple 平台上执行：

1. **架构检测**：如果 `CMAKE_OSX_ARCHITECTURES` 未被用户预先设置，则通过执行 `uname -m` 命令自动检测当前主机的 CPU 架构，并将结果写入缓存变量
2. **部署目标**：如果 `CMAKE_OSX_DEPLOYMENT_TARGET` 未被设置，则默认设为 `11.2`（对应 macOS Big Sur 11.2），确保生成的二进制文件兼容该版本及以上的 macOS 系统

### 平台条件

- 该文件的所有逻辑均包含在 `if(APPLE)` 条件块中
- Windows 和 Linux 平台不受该文件影响

## 依赖关系

- 无外部 cmake 文件依赖
- 依赖系统命令 `uname`（仅 macOS）
