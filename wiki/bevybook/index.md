# Bevy 完整教程
欢迎来到 Bevy 游戏引擎完整教程！本教程旨在帮助你从零开始掌握 Bevy 游戏开发。

## 教程结构

本教程采用渐进式学习路径，从基础概念到高级应用，帮助你系统地学习 Bevy。

### 学习路径

```
Foundation（基础）
    ↓
ECS（实体组件系统）
    ↓
Assets（资源管理）
    ↓
Input（输入处理）
    ↓
Graphics（图形渲染）
    ├── 2D Graphics
    └── 3D Graphics
    ↓
UI & Audio（界面与音频）
    ↓
Architecture（架构设计）
    ↓
Advanced（高级主题）
```

## 目录

### 第一部分：基础（Foundation）

- [快速入门](/wiki/bevybook/foundation/快速入门) - 安装、配置、第一个程序
- [Bevy 与 Rust 框架](/wiki/bevybook/foundation/bevy——rust框架) - Bevy 简介、Rust 基础
- [游戏引擎基础](/wiki/bevybook/foundation/游戏引擎基础) - 游戏引擎核心概念

**索引文件**：[Foundation/README.md](/wiki/bevybook/foundation/)

### 第二部分：ECS（实体组件系统）

ECS 是 Bevy 的核心编程范式，理解 ECS 是掌握 Bevy 的关键。

- [核心编程框架（ECS）](/wiki/bevybook/ecs/核心编程框架（ecs）) - ECS 概述
- [ECS 基础概述](/wiki/bevybook/ecs/) - 组件、实体、系统基础
- [ECS 进阶](/wiki/bevybook/ecs/ecs进阶) - 查询、资源、事件系统

**索引文件**：[ECS/README.md](/wiki/bevybook/ecs/)

### 第三部分：资源管理（Assets）

- [资源管理](/wiki/bevybook/assets/资源管理) - 资源加载、生命周期、异步加载

**索引文件**：[Assets/README.md](/wiki/bevybook/assets/)

### 第四部分：输入处理（Input）

- [输入基础](/wiki/bevybook/input/input基础) - 输入系统概述
- [输入处理](/wiki/bevybook/input/输入处理) - 键盘、鼠标、游戏手柄

**索引文件**：[Input/README.md](/wiki/bevybook/input/)

### 第五部分：图形渲染（Graphics）

#### 2D 图形

- [2D 基础](/wiki/bevybook/2d_graphics/2d基础) - 2D 渲染基础概念
- [2D 开发](/wiki/bevybook/2d_graphics/2d开发) - 精灵、相机、2D 物理

**索引文件**：[2D_Graphics/README.md](/wiki/bevybook/2d_graphics/)

#### 3D 图形

- [3D 开发](/wiki/bevybook/3d_graphics/3d开发) - 3D 模型、材质、光照、相机

**索引文件**：[3D_Graphics/README.md](/wiki/bevybook/3d_graphics/)

### 第六部分：UI、音频与窗口（UI & Audio & Window）

- [窗口管理](/wiki/bevybook/ui_audio_window/窗口) - 窗口创建、配置、多窗口、透明窗口
- [用户界面（UI）](/wiki/bevybook/ui_audio_window/ui) - UI 组件、布局、样式、交互
- [音频系统](/wiki/bevybook/ui_audio_window/音频) - 音频加载、播放控制、3D 音频

**索引文件**：[UI_Audio_Window/README.md](/wiki/bevybook/ui_audio_window/)

### 第七部分：架构设计（Architecture）

- [代码组织](/wiki/bevybook/architecture/代码组织) - 项目结构、模块化设计
- [逻辑-渲染分离](/wiki/bevybook/architecture/逻辑-渲染分离) - MainWorld 与 RenderApp
- [插件系统](/wiki/bevybook/architecture/plugin系统) - 创建插件、插件组、插件管理

**索引文件**：[Architecture/README.md](/wiki/bevybook/architecture/)

### 第八部分：高级主题（Advanced）

- [性能优化](/wiki/bevybook/advanced/性能优化) - 优化技巧、性能分析
- [自定义渲染](/wiki/bevybook/advanced/自定义渲染) - 自定义着色器、渲染管线
- [网络编程](/wiki/bevybook/advanced/网络编程) - 多人游戏、网络同步
- [拆解学习](/wiki/bevybook/advanced/拆解学习) - 深入理解 Bevy 内部机制

**索引文件**：[Advanced/README.md](/wiki/bevybook/advanced/)

### 第九部分：示例项目（Examples）

- [示例项目索引](/wiki/bevybook/examples/) - 完整项目示例

## 快速开始

如果你是 Bevy 新手，建议按以下顺序学习：

1. **Foundation（基础）** - 了解 Bevy 和 Rust 基础
2. **ECS（实体组件系统）** - 掌握 Bevy 的核心编程范式
3. **Assets（资源管理）** - 学习如何加载和管理资源
4. **Input（输入处理）** - 处理用户输入
5. **Graphics（图形渲染）** - 根据你的需求选择 2D 或 3D
6. **UI & Audio & Window（界面、音频与窗口）** - 添加用户界面、音效和窗口管理
7. **Architecture（架构设计）** - 学习如何组织大型项目
8. **Advanced（高级主题）** - 深入高级功能

## 教程特点

- **渐进式学习**：从简单到复杂，循序渐进
- **实用示例**：每个概念都配有实际代码示例
- **中文友好**：全中文教程，降低学习门槛
- **完整覆盖**：涵盖 Bevy 的核心功能和高级特性
- **最佳实践**：分享实际开发中的经验和技巧

## 相关资源

- [Bevy 官方文档](https://bevyengine.org/learn/)
- [Bevy 官方示例](https://github.com/bevyengine/bevy/tree/main/examples)
- [Bevy Cheatbook](https://bevy-cheatbook.github.io/)
- [Bevy Discord](https://discord.gg/bevy)

## 许可证

本教程遵循与 Bevy 相同的许可证。

## 贡献

欢迎提交问题和改进建议！

---

**最后更新**：2025-01-XX


