# procedural.h / procedural.cpp - 程序化生成框架

## 概述

本文件定义了 Cycles 的程序化生成（Procedural）框架，提供在渲染前动态生成场景节点的机制。`Procedural` 是一个抽象基类，派生类可在 `generate()` 中创建几何体、物体等任意节点。`ProceduralManager` 负责在场景更新时协调所有程序化节点的生成过程。

## 类与结构体

### Procedural
- **继承**: `Node`, `NodeOwner`（双继承）
- **功能**: 程序化生成器抽象基类，既是节点又是节点所有者
- **关键成员**:
  - `nodes` — 由此程序化生成器创建的节点列表（`unique_ptr_vector<Node>`）
- **关键方法**:
  - `generate()` — 纯虚函数，入口方法，在每次 ProceduralManager 更新时调用，派生类在此生成场景数据
  - `create_node<T>()` — 模板方法，创建节点并设置 owner 为当前 Procedural
  - `delete_node<T>()` — 模板方法，删除由此 Procedural 创建的节点

### ProceduralManager
- **功能**: 程序化生成管理器，协调所有 Procedural 的更新
- **关键成员**: `need_update_` — 更新标记
- **关键方法**:
  - `update()` — 遍历场景中所有 Procedural 并调用其 `generate()` 方法
  - `tag_update()` — 标记需要更新
  - `need_update()` — 检查是否需要更新

## 核心函数

无独立核心函数。`NODE_ABSTRACT_DEFINE(Procedural)` 宏注册了节点类型 `"procedural_base"`。

## 依赖关系

- **内部头文件**: `graph/node.h`, `util/unique_ptr_vector.h`
- **cpp 额外引用**: `scene/scene.h`, `scene/stats.h`, `util/progress.h`
- **被引用**: `scene/scene.cpp`（Scene 持有 `procedurals` 列表和 `ProceduralManager`）, `scene/shader.cpp`

## 实现细节 / 关键算法

1. **双继承设计**: `Procedural` 同时继承 `Node`（可被场景管理）和 `NodeOwner`（可拥有其他节点），形成层次化的所有权模型。
2. **生成时机**: `ProceduralManager::update()` 在 `Scene::device_update()` 的早期阶段被调用（着色器编译之后、背景更新之前），确保程序化生成的节点能参与后续的完整更新流程。
3. **所有权管理**: `create_node<T>()` 通过 `node->set_owner(this)` 建立所有权关系，节点存储在 Procedural 的 `nodes` 列表中。`delete_node<T>()` 验证所有权后从列表中移除。
4. **可中断**: `update()` 在每个 Procedural 的 `generate()` 调用后检查 `progress.get_cancel()`，支持用户中断长时间的程序化生成。
5. **统计集成**: 使用 `scoped_callback_timer` 记录整体更新时间到 `SceneUpdateStats::procedurals`。

## 关联文件

- `scene/scene.h/.cpp` — 场景管理（持有程序化节点列表和管理器）
- `graph/node.h` — 节点基类（`Node` 和 `NodeOwner` 定义）
- `scene/stats.h/.cpp` — 更新时间统计
