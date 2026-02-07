# ECS（实体组件系统）
ECS 是 Bevy 的核心编程范式。理解 ECS 是掌握 Bevy 的关键。

## 内容列表

### 1. [核心编程框架（ECS）](/wiki/BevyBook/ECS/核心编程框架（ECS）)

- ECS 概述
- 为什么使用 ECS
- ECS 的核心概念
- Bevy ECS 的特点

**学习目标**：理解 ECS 的基本概念和优势

### 2. [组件（Components）](/wiki/BevyBook/ECS/组件（Components）)

- 定义组件
- 组件类型
- 组件生命周期
- 组件存储
- 不可变组件

**学习目标**：能够定义和使用组件

### 3. [实体（Entities）](/wiki/BevyBook/ECS/实体（Entities）)

- 创建实体
- 实体 ID
- 实体操作
- 实体禁用
- 实体层次结构

**学习目标**：能够创建和操作实体

### 4. [命令（Commands）](/wiki/BevyBook/ECS/命令（Commands）)

- Commands 概念
- 创建实体
- 操作实体
- 管理资源
- 注册和运行系统
- 触发事件
- 发送消息
- 实体层次结构

**学习目标**：能够使用 Commands 安全地修改世界

### 5. [系统（Systems）](/wiki/BevyBook/ECS/系统（Systems）)

- 定义系统
- 系统参数
- 系统闭包
- 泛型系统
- 一次性系统
- 系统管道
- 独占系统
- 可失败系统参数
- 系统错误处理

**学习目标**：能够定义和使用系统

### 6. [查询（Queries）](/wiki/BevyBook/ECS/查询（Queries）)

- 基本查询
- 查询过滤器
- 自定义查询参数
- 查询组合
- 并行查询

**学习目标**：能够使用查询访问组件

### 7. [资源（Resources）](/wiki/BevyBook/ECS/资源（Resources）)

- 定义资源
- 访问资源
- 资源初始化
- 资源生命周期
- 资源变更检测

**学习目标**：能够定义和使用资源

### 8. [系统调度（Schedule & App）](/wiki/BevyBook/ECS/系统调度（Schedule与App）)

- Schedule 概念
- 系统执行顺序
- 系统集（SystemSet）
- 自定义 Schedule
- 固定时间步
- 运行条件
- 系统步进
- 非确定性系统顺序

**学习目标**：能够控制系统执行顺序和应用生命周期

### 9. [ECS 进阶](/wiki/BevyBook/ECS/ECS进阶)

- **变更检测（Change Detection）**
  - 检测组件变化
  - 检测资源变化
  - 变更检测方法
  
- **组件生命周期钩子（Component Hooks）**
  - 定义钩子
  - 钩子类型
  - 钩子应用场景
  
- **关系系统（Relationships）**
  - 定义自定义关系
  - 使用关系
  - 关系遍历
  
- **并行查询（Parallel Queries）**
  - 并行迭代器
  - 批处理策略
  - 性能优化
  
- **查询组合（Query Combinations）**
  - 实体间交互
  - 组合查询方法
  
- **观察者模式（Observers）**
  - 响应组件生命周期事件
  - 响应自定义事件
  
- **消息系统（Messages）**
  - 定义消息
  - 发送和接收消息
  
- **错误处理（Error Handling）**
  - 系统错误处理
  - 错误处理器
  
- **动态 ECS（Dynamic ECS）**
  - 动态创建组件
  - 动态创建实体

**学习目标**：掌握 ECS 的高级功能和最佳实践

## 学习建议

1. **理解概念**：ECS 是一种新的编程范式，需要时间适应
2. **多写代码**：通过实践加深理解
3. **阅读示例**：查看 Bevy 官方示例中的 ECS 用法
4. **思考设计**：思考如何用 ECS 思维设计游戏逻辑


## 学习路径

建议按照以下顺序学习：

1. **核心编程框架（ECS）**：理解 ECS 的基本概念和优势
2. **组件（Components）**：学习如何定义和使用组件
3. **实体（Entities）**：学习如何创建和操作实体
4. **命令（Commands）**：学习如何使用 Commands 安全地修改世界
5. **系统（Systems）**：学习如何定义和使用系统
6. **查询（Queries）**：学习如何使用查询访问组件
7. **资源（Resources）**：学习如何定义和使用资源
8. **系统调度（Schedule & App）**：学习如何控制系统执行顺序和应用生命周期
9. **ECS 进阶**：掌握高级功能和最佳实践

每个章节都基于 Bevy 官方示例编写，确保内容的准确性和实用性。

## 相关资源

- [Bevy ECS 官方文档](https://docs.rs/bevy_ecs/latest/bevy_ecs/)
- [ECS 设计模式](https://en.wikipedia.org/wiki/Entity_component_system)
- [Bevy ECS 示例](https://github.com/bevyengine/bevy/tree/main/examples/ecs)

## 下一步

完成本部分学习后，建议继续学习：

- [Assets（资源管理）](/wiki/BevyBook/Assets/) - 学习如何加载和管理资源
- [Input（输入处理）](/wiki/BevyBook/Input/) - 处理用户输入

---

**索引**：[返回主目录](/wiki/BevyBook/README)

