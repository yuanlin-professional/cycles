# cycles_xml.h / cycles_xml.cpp - XML 场景描述文件解析器

## 概述

`cycles_xml.h` 和 `cycles_xml.cpp` 实现了 Cycles 独立渲染器的 XML 场景文件读取功能。该模块将自定义的 XML 场景描述格式解析为 Cycles 内部的场景（Scene）对象，支持相机、着色器图（Shader Graph）、网格（Mesh）、灯光、背景、变换层级、子文件包含等场景元素的完整加载。它是独立渲染器（cycles standalone）的核心场景输入模块。

## 类与结构体

### XMLReadState
- **继承**: `XMLReader`（来自 `graph/node_xml.h`）
- **功能**: XML 解析过程中的递归状态容器，在遍历 XML 节点树时传递和维护当前上下文
- **关键成员**:
  - `scene` (`Scene*`) — 当前场景指针
  - `tfm` (`Transform`) — 当前累积变换矩阵，初始化为单位矩阵
  - `smooth` (`bool`) — 当前法线平滑状态（smooth/flat）
  - `shader` (`Shader*`) — 当前活动着色器
  - `base` (`string`) — 当前文件所在目录的基路径，用于解析相对路径
  - `dicing_rate` (`float`) — 当前细分速率，默认 1.0
  - `object` (`Object*`) — 当前对象指针

## 枚举与常量

### 宏定义（cycles_xml.h）
- `RAD2DEGF(_rad)` — 弧度转角度
- `DEG2RADF(_deg)` — 角度转弧度

## 核心函数

### xml_read_file()
- **签名**: `void xml_read_file(Scene *scene, const char *filepath)`
- **功能**: 模块的公共入口函数（在头文件中声明）。初始化 `XMLReadState`，设置默认着色器和变换，调用 `xml_read_include` 加载指定的 XML 文件，最后将场景的层次包围体（BVH）类型设为 `BVH_TYPE_STATIC`

### XML 属性读取辅助函数
- `xml_read_int()` — 从 XML 节点读取整数属性
- `xml_read_int_array()` — 从 XML 节点读取整数数组（空格分隔）
- `xml_read_float()` — 从 XML 节点读取浮点数属性
- `xml_read_float_array()` — 从 XML 节点读取浮点数组
- `xml_read_float3()` — 从 XML 节点读取三维向量
- `xml_read_float3_array()` — 从 XML 节点读取三维向量数组
- `xml_read_float4()` — 从 XML 节点读取四维向量
- `xml_read_string()` — 从 XML 节点读取字符串属性
- `xml_equal_string()` — 比较 XML 属性值（大小写不敏感）

### xml_read_camera()
- **签名**: `static void xml_read_camera(XMLReadState &state, const xml_node node)`
- **功能**: 解析 `<camera>` 节点。设置分辨率宽高，通过 `xml_read_node` 读取通用属性，设置相机变换矩阵并触发更新

### xml_read_shader_graph()
- **签名**: `static void xml_read_shader_graph(XMLReadState &state, Shader *shader, const xml_node graph_node)`
- **功能**: 解析着色器图定义。遍历子节点创建着色器节点（ShaderNode），处理 `<connect>` 节点建立输入/输出插口连接。支持 OSL 着色器节点（通过 `osl_shader` 标签加载 `.osl` 文件）。特殊处理 `image_texture` 和 `environment_texture` 节点的文件路径（与基路径拼接）。最终将着色器图设置到 Shader 对象并触发更新

### xml_read_shader()
- **签名**: `static void xml_read_shader(XMLReadState &state, const xml_node node)`
- **功能**: 创建新的 `Shader` 节点并调用 `xml_read_shader_graph` 解析其着色器图

### xml_read_background()
- **签名**: `static void xml_read_background(XMLReadState &state, const xml_node node)`
- **功能**: 解析 `<background>` 节点，同时配置背景设置和背景着色器图

### xml_read_mesh()
- **签名**: `static void xml_read_mesh(const XMLReadState &state, const xml_node node)`
- **功能**: 解析 `<mesh>` 节点，构建网格几何体。支持两种模式：
  - **普通网格**: 从顶点（P）、顶点索引（verts）、面顶点数（nverts）数据构建三角形网格，支持顶点法线（VN）、UV 坐标、UV 切线和切线符号等属性
  - **细分曲面**: 支持 Catmull-Clark 和 Linear 两种细分模式，保存原始面数据用于细分

### xml_read_light()
- **签名**: `static void xml_read_light(XMLReadState &state, const xml_node node)`
- **功能**: 解析 `<light>` 节点，创建灯光对象及其关联的 Object，设置变换和可见性（排除相机射线）

### xml_read_transform()
- **签名**: `static void xml_read_transform(const xml_node node, Transform &tfm)`
- **功能**: 解析变换属性，支持四种变换操作（按属性存在顺序依次应用）：
  - `matrix` — 4x4 变换矩阵（16个浮点数）
  - `translate` — 平移向量
  - `rotate` — 旋转（角度 + 轴向量）
  - `scale` — 缩放向量

### xml_read_state()
- **签名**: `static void xml_read_state(XMLReadState &state, const xml_node node)`
- **功能**: 解析 `<state>` 节点，更新当前渲染状态：查找并设置引用的着色器和对象，读取细分速率和插值模式（smooth/flat）

### xml_read_object()
- **签名**: `static void xml_read_object(XMLReadState &state, const xml_node node)`
- **功能**: 解析 `<object>` 节点，创建新的 Mesh 和 Object 对象并读取属性

### xml_read_scene()
- **签名**: `static void xml_read_scene(XMLReadState &state, const xml_node scene_node)`
- **功能**: 场景根节点递归遍历函数。按节点名称分发到对应的解析函数，支持的节点类型：`film`、`integrator`、`camera`、`shader`、`background`、`mesh`、`light`、`transform`、`state`、`include`、`object`。`transform` 和 `state` 节点会创建子状态并递归解析其子节点

### xml_read_include()
- **签名**: `static void xml_read_include(XMLReadState &state, const string &src)`
- **功能**: 加载并解析一个外部 XML 文件。使用 pugixml 库解析文件，以包含文件的目录作为新的基路径创建子状态，然后查找 `<cycles>` 根节点并递归调用 `xml_read_scene`

## 依赖关系

### 内部头文件
- `graph/node_xml.h` — 通用节点 XML 读取基础设施（`XMLReader`、`xml_read_node`）
- `scene/background.h` — 背景节点
- `scene/camera.h` — 相机节点
- `scene/film.h` — 胶片/成像平面节点
- `scene/integrator.h` — 积分器节点
- `scene/light.h` — 灯光节点
- `scene/mesh.h` — 网格几何体
- `scene/object.h` — 场景对象
- `scene/osl.h` — OSL 着色器管理
- `scene/scene.h` — 场景管理
- `scene/shader.h` — 着色器
- `scene/shader_graph.h` — 着色器图
- `scene/shader_nodes.h` — 着色器节点类型
- `util/log.h` — 日志系统
- `util/path.h` — 路径工具
- `util/projection.h` — 投影变换
- `util/string.h` — 字符串工具
- `util/transform.h` — 变换矩阵
- `util/xml.h` — pugixml 封装

### 外部库
- pugixml（通过 `util/xml.h`）— XML 解析库

### 被引用
- `src/app/cycles_standalone.cpp` — 通过 `xml_read_file()` 加载 XML 场景

## 实现细节 / 关键算法

1. **递归状态传递**: 使用 `XMLReadState` 结构体在 XML 树的递归遍历中传递上下文。`<transform>` 和 `<state>` 节点会创建状态副本（substate），实现变换和属性的层级继承——子节点继承父节点的变换矩阵、着色器、平滑状态等。

2. **着色器图构建**: 着色器图的解析分两步进行：首先创建所有着色器节点并读取其属性，然后处理 `<connect>` 节点建立节点间的连接。节点查找通过 `graph_reader.node_map`（基于 `ustring` 的映射表）实现。

3. **网格三角化**: 对于非细分网格，多边形（n-gon）通过扇形三角化（fan triangulation）转换为三角形：以第一个顶点为中心，依次连接后续相邻顶点对。

4. **相对路径解析**: 使用 `state.base` 存储当前文件的目录路径，所有相对路径（纹理文件、包含文件、OSL 脚本）都与其拼接为绝对路径。`<include>` 嵌套时会更新子状态的基路径。

5. **BVH 类型**: 通过 XML 加载的场景默认使用 `BVH_TYPE_STATIC`（静态层次包围体），因为独立渲染器中的场景不需要动态更新。

## 关联文件

- `src/app/cycles_standalone.cpp` — 调用 `xml_read_file()` 的主程序
- `src/graph/node_xml.h` — 提供 `XMLReader` 基类和 `xml_read_node` 通用属性读取
- `src/util/xml.h` — pugixml 类型别名定义
- `src/app/io_export_cycles_xml.py` — 对应的 Blender 导出脚本（Python），可将 Blender 场景导出为此模块能读取的 XML 格式
