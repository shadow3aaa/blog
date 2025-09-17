---
title: "Tessera UI v2.0.0"
date: 2025-09-17T18:05:00+08:00
---

Tessera UI 是一个基于 Rust 和 wgpu 的，gpu 优先的即时模式 UI 框架。<!--more-->

## 更新内容

v2.0.0 的更新内容如下

### 官网

Tessera UI 现在有了自己的主页 [tessera-ui.github.io](https://tessera-ui.github.io)。这里包含了文档、指南和示例。

### 分片(shard)与导航

shard 是 v2.0.0 中引入的全新特性，用于实现更方便的页面化组件和导航功能。

详细说明见[文档-分片与导航](https://tessera-ui.github.io/guide/shard.html)。

### 渲染器

- **脏矩形检测**: 实现了脏矩形（Dirty Frame）检测机制。当界面无变化时，渲染器会跳过大部分 GPU 工作，大幅降低静态画面的 CPU 和 GPU 占用。
- **指令重排序与批处理**: 引入了基于依赖图的渲染指令重排序系统，智能地对指令进行分组，最大限度地合并渲染批次，减少昂贵的状态切换。多个渲染管线（`Shape`, `Text`, `FluidGlass`）已重构以支持实例化批处理渲染。
- **局部渲染与计算**: 实现了绘制指令的裁剪（Scissor），对于毛玻璃等特效，渲染器会自动计算并应用最小渲染区域。计算着色器（如模糊）现在也可以在指定的局部区域内执行，优化了局部特效的性能。
- **文本渲染优化**: 为文本数据 (`TextData`) 添加了 LRU 缓存，避免对相同文本进行重复的布局和构建，提升了渲染效率。
- **组件内容裁剪**: 实现了核心的组件内容裁剪功能，子组件的绘制不会超出父组件的边界。
- **管线调度优化**: 渲染管线的调度从 O(n) 的线性查找优化为 O(1) 的哈希表查找，加快了指令分派速度。
- **渲染管线 API 变更**: 自定义渲染管线的 `draw` 方法签名已更新，现在需要接收一个裁剪矩形 (`clip_rect`) 参数。
- **图像数据共享**: `image` 组件的 API 现在接收 `Arc<ImageData>` 而不是所有权数据，以支持数据共享。

### 布局

- **容器 API 变更**: `column`, `row`, `boxed` 等容器组件废除了旧的宏和 Trait API，统一使用更灵活的作用域闭包 API (`scope.child(...)`)。
- **尺寸单位变更**: 组件的 `width` 和 `height` 字段不再是 `Option<DimensionValue>`，而是 `DimensionValue`，必须显式提供值或依赖其默认值 (`WRAP`)。
- **布局约束强化**: 在有限的父约束下使用 `DimensionValue::Fill` 时，如果没有提供 `max` 值，现在会触发 `panic`，以防止不明确的布局行为。
- **圆角矩形增强**: `RoundedRectangle` 的角半径单位从像素 (`f32`) 改为 `Dp`，并支持为四个角设置独立的半径值。

### 事件处理

- **API 命名重构**: 整个事件处理相关的 API (如 `StateHandlerFn`, `StateHandlerInput`) 已从 `state_handler` 重命名为 `input_handler`，语义更清晰。
- **剪贴板抽象**: 在核心库 `tessera-ui` 中添加了剪贴板抽象，并为 Android 平台提供了原生支持。
- **API 改进**: 统一了 `cursor_position` API，现在分为相对 (`rel`) 和绝对 (`abs`) 位置。`StateHandlerInput` 添加了事件阻塞方法。
- **问题修复**: 修复了当父节点被视口剔除时，子节点的绝对位置未计算导致事件处理时 `panic` 的严重问题。

### 组件库

- **新增组件**:
  - `SideBar`: 可从侧边滑出的侧边栏，支持毛玻璃和材质背景。
  - `BottomSheet`: 可从底部滑出的底栏，支持毛玻璃和材质背景。
  - `Tabs`: 标签页组件，支持内容切换和指示器动画。
  - `BottomNavBar`: 底部导航栏，支持路由切换和带动画的选择指示器。
  - `ScrollBar`: 可复用的滚动条，支持拖拽、点击和悬浮动画，并已集成到 `scrollable` 中。
  - `Glass Progress`: 带有毛玻璃效果的进度条。
- **组件状态管理重构**: `Tabs`, `Switch`, `GlassSwitch`, `Checkbox` 等多个组件的状态管理被重构，不再使用内部状态，而是要求传入一个由调用方拥有的外部状态 (`Arc<RwLock<...State>>`)，实现了状态的完全解耦。
- **组件改进**:
  - `Scrollable`: 增加了 `Overlay` (悬浮) 和 `Alongside` (并排) 两种滚动条布局，以及 `AlwaysVisible`, `AutoHide`, `Hidden` 三种行为模式。
  - `Dialog`: 集成了 `DialogProvider` 统一 API，遮罩层支持 `Glass` 和 `Material` 风格，并增加了内容淡入淡出动画。
  - `Button` 和 `Surface`: 添加了可配置的阴影属性。
  - `TextEditor`: 增加了 `on_change` 回调并实现了平滑的像素级滚动。
  - `Switch`: 动画曲线改为更平滑的 `ease-in-out` 效果。
  - `FluidGlass`: 增强了边框的高光效果，模拟 3D 斜面感。
- **组件可见性调整**: 部分本应为私有的组件现在已设为私有（如 `cursor.rs`），以防止误用。

### 其他改进

- 重新设计了 Logo 用于主页和文档
- 依赖更新
- 重构和清理代码

## 关于 Tessera UI

关于 Tessera UI 本身的进一步介绍，请参见 [指南-什么是 Tessera UI](https://tessera-ui.github.io/guide/what-is-tessera.html)。
