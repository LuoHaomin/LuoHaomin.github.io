# 实体（Entities）## 概述

**学习目标**：
- 理解实体的概念和作用
- 掌握如何创建和操作实体
- 了解实体 ID 的使用方法
- 理解实体禁用和层次结构

**前置知识要求**：
- 核心编程框架（ECS）
- 组件（Components）
- Rust 基础语法

## 核心概念

### 什么是实体？

实体是具有唯一 ID 的组件集合。实体本身不包含任何数据，它只是一个标识符，用于将组件组合在一起。

**为什么使用实体？**

1. **灵活组合**：实体可以灵活组合不同的组件
2. **唯一标识**：每个实体都有唯一的 ID
3. **高效查询**：通过实体 ID 可以快速访问组件

### 实体的设计思想

实体采用数据导向的设计思想，将数据（组件）和标识符（实体）分离。这种设计使得：

- 实体可以灵活组合不同的组件
- 系统可以高效查询具有特定组件的实体
- 实体可以动态添加和移除组件

## 基础用法

### 创建实体

使用 `Commands` 创建实体并添加组件。关于 Commands 的详细使用方法，请参考[命令（Commands）](/wiki/bevybook/ecs/命令（Commands）)章节。

**源代码文件**：`bevy/examples/ecs/ecs_guide.rs`

**代码示例**：

```rust
// 这是一个"启动"系统，在应用启动时恰好运行一次
// 启动系统通常用于创建游戏的初始"状态"
fn startup_system(mut commands: Commands, mut game_state: ResMut<GameState>) {
    // 向我们的世界添加一些玩家。玩家从分数 0 开始
    commands.spawn_batch(vec![
        (
            Player {
                name: "Alice".to_string(),
            },
            Score { value: 0 },
            PlayerStreak::None,
        ),
        (
            Player {
                name: "Bob".to_string(),
            },
            Score { value: 0 },
            PlayerStreak::None,
        ),
    ]);

    // 设置总玩家数为 2
    game_state.total_players = 2;
}
```

**关键要点**：
- 使用 `Commands` 创建实体
- 使用 `spawn()` 创建单个实体
- 使用 `spawn_batch()` 批量创建实体
- 可以在创建实体时同时添加多个组件

**说明**：
实体通过 Commands 创建。Commands 提供了一种安全的方式来修改世界，允许系统并行执行。关于 Commands 的详细使用方法，请参考[命令（Commands）](/wiki/bevybook/ecs/命令（Commands）)章节。

### 实体 ID

每个实体都有唯一的 ID，可以通过 `.id()` 方法获取。

**源代码文件**：`bevy/examples/ecs/ecs_guide.rs`

**代码示例**：

```rust
// 调用 `.id()` 在生成实体后将返回生成实体的 `Entity` 标识符
// 即使实体本身尚未在世界中实例化
// 这有效是因为 Commands 会在实际生成实体之前通过原子计数器保留实体 ID
let alice = commands.spawn(Name::new("Alice")).id();
let bob = commands.spawn((Name::new("Bob"), Targeting(alice))).id();
```

**关键要点**：
- 使用 `.id()` 方法获取实体 ID
- 实体 ID 在实体创建之前就可以获取
- 实体 ID 是唯一的，可以用于引用实体
- 实体 ID 可以存储在组件中，用于建立实体间的关系

**说明**：
实体 ID 是实体的唯一标识符。即使在实体创建之前，也可以通过 `Commands` 获取实体 ID。这使得可以在创建实体时建立实体间的关系。

## 进阶用法

### 实体禁用

实体禁用允许你隐藏实体而不删除它们。这对于实现"休眠"对象或管理网络实体很有用。

**源代码文件**：`bevy/examples/ecs/entity_disabling.rs`

**代码示例**：

```rust
use bevy::ecs::entity_disabling::Disabled;
use bevy::prelude::*;

fn disable_entities_on_click(
    click: On<Pointer<Click>>,
    valid_query: Query<&DisableOnClick>,
    mut commands: Commands,
) {
    if valid_query.contains(click.entity) {
        // 只需将 `Disabled` 组件添加到实体即可禁用它
        // 注意 `Disabled` 组件只添加到实体本身，其子实体不受影响
        commands.entity(click.entity).insert(Disabled);
    }
}

// 这里的查询不会找到具有 `Disabled` 组件的实体
// 因为它没有明确包含它
fn list_all_named_entities(
    query: Query<&Name>,
    mut name_text_query: Query<&mut Text, With<EntityNameText>>,
    mut commands: Commands,
) {
    // 查询会自动跳过被禁用的实体
    for name in query.iter().sort::<&Name>() {
        // ...
    }
}

fn reenable_entities_on_space(
    mut commands: Commands,
    // 这个查询可以找到被禁用的实体
    // 因为它明确包含了 `Disabled` 组件
    disabled_entities: Query<Entity, With<Disabled>>,
    input: Res<ButtonInput<KeyCode>>,
) {
    if input.just_pressed(KeyCode::Space) {
        for entity in disabled_entities.iter() {
            // 要重新启用实体，只需移除 `Disabled` 组件
            commands.entity(entity).remove::<Disabled>();
        }
    }
}
```

**关键要点**：
- 使用 `Disabled` 组件禁用实体
- 被禁用的实体会被默认查询过滤器跳过
- 可以通过明确包含 `Disabled` 组件来查询被禁用的实体
- 禁用实体不会影响其子实体

**注意事项**：
- 禁用实体会使它们对 ECS 不可见，但这不是其主要目的
- `Visibility` 应该用于隐藏实体
- 被禁用的实体会被完全跳过，这可能导致微妙的错误

**最佳实践**：
- 使用 `Visibility` 隐藏实体
- 使用 `Disabled` 禁用不需要处理的实体
- 注意禁用实体对查询的影响

### 实体层次结构

实体可以形成层次结构，通过 `ChildOf` 和 `Children` 组件实现父子关系。

**源代码文件**：`bevy/examples/ecs/hierarchy.rs`

**代码示例**：

```rust
fn setup(mut commands: Commands, asset_server: Res<AssetServer>) {
    commands.spawn(Camera2d);
    let texture = asset_server.load("branding/icon.png");

    // 生成一个没有父实体的根实体
    let parent = commands
        .spawn((
            Sprite::from_image(texture.clone()),
            Transform::from_scale(Vec3::splat(0.75)),
        ))
        // 以该实体作为父实体，运行一个 lambda 来生成其子实体
        .with_children(|parent| {
            // parent 是一个 ChildSpawnerCommands，具有与 Commands 类似的 API
            parent.spawn((
                Transform::from_xyz(250.0, 0.0, 0.0).with_scale(Vec3::splat(0.75)),
                Sprite {
                    image: texture.clone(),
                    color: BLUE.into(),
                    ..default()
                },
            ));
        })
        .id();

    // 另一种方法是使用 add_child 函数在父实体已经生成后添加子实体
    let child = commands
        .spawn((
            Sprite {
                image: texture,
                color: LIME.into(),
                ..default()
            },
            Transform::from_xyz(0.0, 250.0, 0.0).with_scale(Vec3::splat(0.75)),
        ))
        .id();

    // 将子实体添加到父实体
    commands.entity(parent).add_child(child);
}

// 一个简单的系统来旋转根实体，并分别旋转其所有子实体
fn rotate(
    mut commands: Commands,
    time: Res<Time>,
    mut parents_query: Query<(Entity, &Children), With<Sprite>>,
    mut transform_query: Query<&mut Transform, With<Sprite>>,
) {
    for (parent, children) in &mut parents_query {
        if let Ok(mut transform) = transform_query.get_mut(parent) {
            transform.rotate_z(-PI / 2. * time.delta_secs());
        }

        // 要遍历实体的子实体，只需将 Children 组件视为 Vec
        // 或者，你可以查询具有 ChildOf 组件的实体
        for child in children {
            if let Ok(mut transform) = transform_query.get_mut(*child) {
                transform.rotate_z(PI * time.delta_secs());
            }
        }
    }
}
```

**关键要点**：
- 使用 `with_children()` 方法在创建实体时添加子实体
- 使用 `add_child()` 方法在实体创建后添加子实体
- 使用 `Children` 组件访问子实体列表
- 使用 `ChildOf` 组件访问父实体
- 当添加 `DefaultPlugins` 时，系统会自动传播 `Transform` 和 `Visibility` 从父实体到子实体

**注意事项**：
- 层次结构通过 `ChildOf` 和 `Children` 组件实现
- 销毁父实体会自动销毁所有子实体
- 可以通过移除 `ChildOf` 组件来断开父子关系

**最佳实践**：
- 使用层次结构组织相关的实体
- 注意层次结构对变换和可见性传播的影响
- 合理使用层次结构，避免过深的嵌套

## 实际应用

### 在游戏开发中的应用场景

实体在游戏开发中有广泛的应用：

1. **游戏对象**：玩家、敌人、道具等游戏对象都是实体
2. **层次结构**：场景图、UI 层次结构等
3. **实体管理**：实体池、实体禁用等

### 常见问题

**问题 1**：如何获取实体的 ID？

**解决方案**：使用 `.id()` 方法在创建实体时获取 ID，或使用查询获取实体的 ID。

**问题 2**：如何禁用实体而不删除它们？

**解决方案**：使用 `Disabled` 组件禁用实体。被禁用的实体会被默认查询过滤器跳过。

**问题 3**：如何建立实体间的层次结构？

**解决方案**：使用 `with_children()` 或 `add_child()` 方法建立父子关系。Bevy 会自动管理 `ChildOf` 和 `Children` 组件。

### 性能考虑

1. **实体创建**：批量创建实体比逐个创建更高效
2. **实体查询**：使用查询过滤器减少查询的实体数量
3. **实体禁用**：禁用不需要处理的实体可以提高性能

## 相关资源

**相关源代码文件**：
- `bevy/examples/ecs/ecs_guide.rs` - ECS 完整指南示例（实体创建）
- `bevy/examples/ecs/entity_disabling.rs` - 实体禁用示例
- `bevy/examples/ecs/hierarchy.rs` - 实体层次结构示例

**官方文档链接**：
- [Bevy Entity 官方文档](https://docs.rs/bevy_ecs/latest/bevy_ecs/entity/index.html)
- [Entity 禁用文档](https://docs.rs/bevy_ecs/latest/bevy_ecs/entity_disabling/index.html)
- [Hierarchy 文档](https://docs.rs/bevy/latest/bevy/hierarchy/index.html)

**进一步学习建议**：
- 学习系统（Systems），了解如何处理实体
- 学习查询（Queries），了解如何查询实体
- 学习关系系统（Relationships），了解如何建立实体间的关系

---

**索引**：[返回上级目录](/wiki/bevybook/ecs/)


