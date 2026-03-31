# integrator_render_scheduler_test.cpp - 渲染调度器分辨率计算测试

## 概述

此文件测试 Cycles 渲染调度器（Render Scheduler）中与分辨率计算相关的两个辅助函数。渲染调度器负责在交互式渲染中动态调整渲染分辨率，以在响应速度和图像质量之间取得平衡。

## 测试用例

### TEST(IntegratorRenderScheduler, calculate_resolution_divider_for_resolution)
- **功能**: 验证根据目标分辨率计算分辨率除数的函数。该函数根据原始图像尺寸和期望的最大分辨率维度，计算需要的缩小倍数。
- **验证数据**:
  - 1920x1080 图像，目标分辨率 1920 -> 除数 1（不缩小）
  - 1920x1080 图像，目标分辨率 960 -> 除数 2（缩小一半）
  - 1920x1080 图像，目标分辨率 480 -> 除数 4（缩小四倍）

### TEST(IntegratorRenderScheduler, calculate_resolution_for_divider)
- **功能**: 验证根据分辨率除数反向计算实际渲染分辨率的函数。此函数用于在给定除数时确定实际渲染的像素分辨率。
- **验证数据**:
  - 1920x1080 图像，除数 1 -> 分辨率 1440
  - 1920x1080 图像，除数 2 -> 分辨率 720
  - 1920x1080 图像，除数 4 -> 分辨率 360

## 依赖关系
- **被测源文件**: `integrator/render_scheduler.h`
- **测试框架**: Google Test (GTest)

## 关联文件
- `src/integrator/render_scheduler.h` - 渲染调度器声明
- `src/integrator/render_scheduler.cpp` - 渲染调度器实现
