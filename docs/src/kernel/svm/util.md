# util.h - SVM 工具函数：栈操作与节点读取

## 概述
`util.h` 提供了着色器虚拟机(SVM)运行时所需的基础工具函数，包括栈的读写操作、SVM 指令节点的读取，以及指令参数的解包函数。这些函数是所有 SVM 节点实现的通用基础设施，每个节点的执行都依赖此文件中的栈操作来获取输入和存储输出。

该文件还提供了 `dPdx` / `dPdy` 辅助函数，用于计算着色点的屏幕空间微分，这在纹理过滤和凹凸映射中不可或缺。

## 核心函数

### 栈读取函数
- **`stack_load_float3(stack, a)`** - 从栈偏移 `a` 处加载 `float3` 向量（连续 3 个 float）
- **`stack_load_float3_default(stack, a, value)`** - 带默认值的 float3 加载，当 `a == SVM_STACK_INVALID` 时返回默认值
- **`stack_load_float(stack, a)`** - 从栈偏移 `a` 处加载单个 float
- **`stack_load_float_default(stack, a, value)`** - 带默认值的 float 加载（两个重载版本：`uint` 编码的浮点默认值和直接 `float` 默认值）
- **`stack_load_int(stack, a)`** - 从栈偏移 `a` 处加载整数（通过 `__float_as_int` 类型双关）
- **`stack_load_int_default(stack, a, value)`** - 带默认值的整数加载

### 栈写入函数
- **`stack_store_float3(stack, a, f)`** - 将 `float3` 写入栈偏移 `a`
- **`stack_store_float(stack, a, f)`** - 将 `float` 写入栈偏移 `a`
- **`stack_store_int(stack, a, i)`** - 将整数写入栈偏移 `a`（通过 `__int_as_float` 编码）

### 栈验证
- **`stack_valid(a)`** - 检查栈偏移量是否有效（非 `SVM_STACK_INVALID`），用于判断输出端口是否被连接

### 节点读取函数
- **`read_node(kg, offset)`** - 从 SVM 节点纹理读取一个 `uint4` 指令并递增偏移量
- **`read_node_float(kg, offset)`** - 读取一个 `uint4` 并将其 4 个分量转换为 `float4`
- **`fetch_node_float(kg, offset)`** - 读取指定偏移处的 `float4`（不修改偏移量）

### 参数解包函数
- **`svm_unpack_node_uchar2(i, x, y)`** - 从一个 `uint` 中解包 2 个 `uchar`（各 8 位）
- **`svm_unpack_node_uchar3(i, x, y, z)`** - 从一个 `uint` 中解包 3 个 `uchar`
- **`svm_unpack_node_uchar4(i, x, y, z, w)`** - 从一个 `uint` 中解包 4 个 `uchar`

### 微分辅助函数
- **`dPdx(sd)`** - 计算着色点在屏幕 X 方向的位置微分
- **`dPdy(sd)`** - 计算着色点在屏幕 Y 方向的位置微分

## 依赖关系
- **内部头文件**:
  - `kernel/globals.h` - `KernelGlobals` 定义
  - `kernel/types.h` - 内核数据类型（`ShaderData` 等）
  - `kernel/svm/types.h` - `SVM_STACK_SIZE`, `SVM_STACK_INVALID` 等常量
- **被引用**: 几乎所有 `kernel/svm/*.h` 节点实现文件，包括 `svm.h`, `noisetex.h`, `voronoi.h`, `brick.h`, `checker.h`, `gradient.h`, `magic.h`, `wave.h`, `sky.h`, `gabor.h` 等

## 实现细节

### 栈布局
SVM 栈是一个固定大小（255 个 float）的本地数组。每个 float 占一个槽位，float3 占 3 个连续槽位。栈偏移量使用 `uchar`（0-254）编码，值 255 (`SVM_STACK_INVALID`) 表示无效。所有栈访问都带有 `kernel_assert` 边界检查。

### 参数打包策略
SVM 编译器将多个小参数（如栈偏移量）打包到单个 `uint` 中。`svm_unpack_node_uchar4` 通过位移和掩码操作将一个 `uint` 拆分为 4 个字节，最大化了每条 `uint4` 指令中可携带的参数数量。

### 类型双关编码
整数通过 `__int_as_float` / `__float_as_int` 存储在浮点栈中，浮点默认值通过 `__uint_as_float` 从指令中解码。这种位级类型双关在 GPU 上是零成本操作。

### 微分计算
`dPdx`/`dPdy` 使用链式法则：`dP/dx = dP/du * du/dx + dP/dv * dv/dx`，其中 `dP/du`、`dP/dv` 是参数空间微分，`du/dx`、`dv/dx` 是屏幕空间到参数空间的映射。

## 关联文件
- `kernel/svm/types.h` - 栈常量定义
- `kernel/svm/svm.h` - 主解释器循环（调用此处的 `read_node` 和栈函数）
- `kernel/globals.h` - `kernel_data_fetch(svm_nodes, ...)` 节点数据获取
