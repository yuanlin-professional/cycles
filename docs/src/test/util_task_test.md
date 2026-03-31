# util_task_test.cpp - 任务调度系统测试

## 概述

此文件测试 Cycles 工具库中的任务调度系统（`TaskScheduler` 和 `TaskPool`）。任务调度系统是 Cycles 渲染器的并行计算基础设施，负责将渲染工作分配到多个线程上执行。

## 测试用例

### TEST(util_task, basic)
- **功能**: 验证任务调度的基本功能。初始化调度器，创建任务池，提交 100 个空任务（`task_run` 函数体为空），等待全部完成后验证处理的任务数量为 100。
- **验证逻辑**:
  1. `TaskScheduler::init(0)` - 初始化调度器（参数 0 表示自动检测线程数）
  2. 向 `TaskPool` 推送 100 个 lambda 任务
  3. `pool.wait_work(&summary)` - 等待所有任务完成
  4. 验证 `summary.num_tasks_handled == 100`
  5. `TaskScheduler::exit()` - 关闭调度器

### TEST(util_task, multiple_times)
- **功能**: 验证任务调度系统在重复初始化/销毁场景下的稳定性。重复 1000 次执行与 `basic` 相同的流程（每次初始化、提交 100 个任务、等待、销毁），确保调度器的生命周期管理没有内存泄漏或竞态条件。
- **验证逻辑**: 每一轮都验证 `summary.num_tasks_handled == 100`

## 依赖关系
- **被测源文件**: `util/task.h`
- **测试框架**: Google Test (GTest)

## 关联文件
- `src/util/task.h` - TaskScheduler、TaskPool 类声明
- `src/util/task.cpp` - 任务调度系统实现
