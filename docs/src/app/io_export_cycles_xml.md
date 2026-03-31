# io_export_cycles_xml.py - Blender 场景到 Cycles XML 格式的网格导出插件

## 概述

`io_export_cycles_xml.py` 是一个 Blender Python 插件（Addon），用于将当前活动对象的网格数据导出为 Cycles 独立渲染器可读取的 XML 场景文件（`.xml`）。该脚本主要面向测试用途，而非最终用户。它通过 Blender 的 `bpy` API 获取网格顶点、面片和 UV 数据，将其序列化为 XML 元素并格式化输出。

导出的 XML 文件可由 Cycles 独立渲染器的 XML 解析模块（`cycles_xml.cpp`）读取，形成完整的 XML 场景导入/导出工作流。

## 核心函数/类

### `strip(root)`
- **功能**: 递归清除 XML 元素树中所有节点的 `text` 和 `tail` 属性，移除空白文本内容，为后续 `minidom` 美化输出做准备
- **参数**: `root` — `xml.etree.ElementTree.Element` 根节点

### `write(node, fname)`
- **功能**: 将 XML 节点树序列化为格式化字符串并写入文件
- **流程**:
  1. 调用 `strip()` 清除空白
  2. 使用 `etree.tostring()` 序列化为字节串
  3. 使用 `minidom.parseString().toprettyxml()` 格式化为可读 XML
  4. 写入指定文件路径

### `CyclesXMLSettings`
- **继承**: `bpy.types.PropertyGroup`
- **功能**: 在 Blender 场景（`bpy.types.Scene`）上注册 `cycles_xml` 属性组，提供导出文件路径配置
- **属性**:
  - `filepath` (`StringProperty`) — 导出 `.xml` 文件的目标路径，最大长度 256

### `RenderButtonsPanel`
- **功能**: 面板基类，限定仅在渲染引擎为 `CYCLES` 时显示
- **位置**: 属性编辑器（Properties）> 渲染（Render）面板

### `PHYSICS_PT_fluid_export`
- **继承**: `RenderButtonsPanel`, `bpy.types.Panel`
- **功能**: 在渲染面板中绘制 "Cycles XML Exporter" UI，提供导出操作按钮

### `ExportCyclesXML`
- **继承**: `bpy.types.Operator`, `ExportHelper`
- **标识符**: `export_mesh.cycles_xml`
- **功能**: 核心导出操作符，执行网格数据提取和 XML 生成
- **导出数据**:
  - `P` — 顶点坐标列表（`float3` 格式，空格分隔）
  - `nverts` — 每个面片的顶点数
  - `verts` — 面片顶点索引
  - `UV` — 逐面片顶点 UV 坐标
- **前提条件**: 场景中必须存在活动对象，且该对象包含网格数据

### `register()` / `unregister()`
- **功能**: Blender 插件标准注册/注销入口，调用 `bpy.utils.register_module()` / `unregister_module()`

## 依赖关系

- **Python 标准库**:
  - `xml.etree.ElementTree` — XML 元素树构建与序列化
  - `xml.dom.minidom` — XML 美化格式输出
- **Blender API**:
  - `bpy` — Blender Python 核心 API
  - `bpy_extras.io_utils.ExportHelper` — 文件导出对话框辅助类
  - `bpy.props.PointerProperty`, `StringProperty` — 属性定义
- **被引用**: 作为 Blender 插件独立运行，可通过 `__main__` 入口直接注册

## 实现细节

1. **网格数据获取**: 使用 `object.to_mesh(scene, True, 'PREVIEW')` 获取应用修改器后的预览网格，包括经过 `tessface` 三角化/四边形化处理的面片数据
2. **UV 处理**: 从 `tessface_uv_textures.active.data` 获取活跃 UV 层的逐面片 UV 坐标，支持三角面（3 UV 对）和四边面（4 UV 对）
3. **XML 结构**: 导出为单个 `<mesh>` 元素，属性包含所有几何数据，符合 Cycles XML 场景格式规范
4. **格式化输出**: 先通过 `strip()` 清除 ElementTree 自动添加的空白，再由 `minidom` 重新生成缩进格式，确保输出可读
5. **注意事项**: 该脚本使用了 Blender 旧版 API（`tessfaces`、`register_module`），可能与较新版本的 Blender 不兼容

## 关联文件

- `src/app/cycles_xml.h` / `src/app/cycles_xml.cpp` — Cycles 独立渲染器的 XML 场景读取器（导入端）
- `src/scene/mesh.h` — Cycles 网格数据结构定义
