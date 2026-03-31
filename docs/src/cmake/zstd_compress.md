# zstd_compress.cpp - GPU 内核二进制文件的 Zstd 压缩工具

## 概述

`zstd_compress.cpp` 是一个独立的命令行工具程序，用于将编译生成的 GPU 内核二进制文件（CUDA `.cubin`、HIP `.fatbin`、OptiX `.ptx` 等）进行 Zstd 压缩。该工具在 CMake 构建流程中被调用，以减小 Cycles 发布包中 GPU 内核文件的体积。压缩后的内核文件在运行时由 Cycles 设备层解压并加载到 GPU 上执行。

## 核心函数/类

### `main(argc, argv)`
- **签名**: `int main(const int argc, const char **argv)`
- **参数**:
  - `argv[1]` — 输入文件路径（待压缩的原始二进制文件）
  - `argv[2]` — 输出文件路径（压缩后的目标文件）
- **返回值**: 成功返回 `0`，任何错误返回 `-1`
- **处理流程**:
  1. 参数检查：要求至少 3 个参数（程序名 + 输入路径 + 输出路径）
  2. 以二进制模式打开输入和输出文件流
  3. 通过 `seekg` 到文件末尾获取输入文件大小
  4. 将整个输入文件读入内存缓冲区（`std::vector<char>`）
  5. 调用 `ZSTD_compressBound()` 计算压缩输出的最大可能大小，分配输出缓冲区
  6. 调用 `ZSTD_compress()` 执行压缩，压缩级别为 **19**（接近最高压缩比）
  7. 将压缩后的数据写入输出文件
- **错误处理**: 每个 I/O 和 Zstd 操作后均检查错误状态，任何失败立即返回 `-1`

## 依赖关系

- **外部库**:
  - `<zstd.h>` — Facebook Zstandard 压缩库，提供 `ZSTD_compress()`、`ZSTD_compressBound()`、`ZSTD_isError()` 等 API
- **C++ 标准库**:
  - `<cstdint>` — 整数类型
  - `<fstream>` — 文件 I/O（`std::ifstream` / `std::ofstream`）
  - `<vector>` — 动态缓冲区
- **被引用**: `src/kernel/CMakeLists.txt` 将其编译为独立可执行文件 `zstd_compress`，并在 CUDA、HIP、HIPRT、OptiX 内核构建的 `add_custom_command` 中调用

## 实现细节

1. **压缩级别**: 使用 Zstd 压缩级别 19（范围 1-22），优先追求高压缩比而非压缩速度。这适用于构建时一次性压缩、运行时多次解压的场景
2. **内存模型**: 采用将整个文件读入内存再一次性压缩的策略，简单直接。对于 GPU 内核二进制文件（通常几 MB 到几十 MB），内存消耗可接受
3. **已知限制**: 代码中的 TODO 注释指出，在 Windows 平台上使用 `std::ifstream(const char*)` 构造函数可能无法正确处理非 ASCII 路径。这是因为 Windows 的文件系统使用宽字符（UTF-16），而标准 C++ 的窄字符文件流未必能正确转换
4. **构建集成**: 在 `CMakeLists.txt` 中，每个 GPU 内核编译目标都有对应的压缩步骤：先编译内核为二进制，再调用 `zstd_compress` 生成 `.compressed` 文件，最终安装的是压缩版本

## 关联文件

- `src/kernel/CMakeLists.txt` — 构建系统中定义 `zstd_compress` 编译目标及其在内核构建流程中的调用
- `src/device/cuda/device.cpp` / `src/device/hip/device.cpp` / `src/device/optix/device.cpp` — 运行时解压并加载压缩内核的设备层实现
