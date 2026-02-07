# 命令（Commands）## 概述

**学习目标**：
- 理解 Commands 的概念和作用
- 掌握如何使用 Commands 修改世界
- 了解 Commands 的各种操作方法
- 理解 Commands 的执行时机

**前置知识要求**：
- 核心编程框架（ECS）
- 组件（Components）
- 实体（Entities）
- 系统（Systems）
- Rust 基础语法


## 核心概念

### 什么是 Commands？

Commands 是一种安全的方式来修改世界（World）。由于系统可以并行执行，直接访问世界是不安全的。Commands 将修改操作排队，在系统执行完成后统一应用。

**为什么使用 Commands？**

1. **线程安全**：Commands 允许系统安全地修改世界，即使系统并行执行
2. **延迟执行**：Commands 将修改操作排队，在系统执行完成后统一应用
3. **命令缓冲**：Commands 提供命令缓冲，允许批量操作

### Commands 的设计思想

Commands 采用命令模式，将修改操作封装为命令，在系统执行完成后统一应用。这种设计使得：

- 系统可以并行执行，同时安全地修改世界
- 修改操作可以延迟执行，提高性能
- 修改操作可以批量处理，减少开销

## 基础用法

### 创建实体

使用 `spawn()` 和 `spawn_batch()` 创建实体。

**源代码文件**：`bevy/examples/ecs/ecs_guide.rs`

**代码示例**：

```rust
fn startup_system(mut commands: Commands, mut game_state: ResMut<GameState>) {
    // 创建游戏规则资源
    commands.insert_resource(GameRules {
        max_rounds: 10,
        winning_score: 4,
        max_players: 4,
    });

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

// 这个系统使用命令缓冲区（可能）在每次迭代时向我们的游戏添加一个新玩家
// 普通系统不能安全地直接访问 World 实例，因为它们并行运行
// 我们的 World 包含所有组件，所以并行修改其任意部分不是线程安全的
// 命令缓冲区使我们能够在不直接访问的情况下排队对 World 的更改
fn new_player_system(
    mut commands: Commands,
    game_rules: Res<GameRules>,
    mut game_state: ResMut<GameState>,
) {
    // 随机添加一个新玩家
    let add_new_player = random::<bool>();
    if add_new_player && game_state.total_players < game_rules.max_players {
        game_state.total_players += 1;
        commands.spawn((
            Player {
                name: format!("Player {}", game_state.total_players),
            },
            Score { value: 0 },
            PlayerStreak::None,
        ));

        println!("Player {} joined the game!", game_state.total_players);
    }
}
```

**关键要点**：
- 使用 `spawn()` 创建单个实体
- 使用 `spawn_batch()` 批量创建实体
- 可以在创建实体时同时添加多个组件
- Commands 将修改操作排队，在系统执行完成后统一应用

**说明**：
Commands 提供了一种安全的方式来修改世界。由于系统可以并行执行，直接访问世界是不安全的。Commands 将修改操作排队，在系统执行完成后统一应用。

### 操作实体

使用 `entity()` 方法获取实体命令，然后可以插入、移除组件或销毁实体。

**源代码文件**：`bevy/examples/ecs/ecs_guide.rs`

**代码示例**：

```rust
// 调用 `.id()` 在生成实体后将返回生成实体的 `Entity` 标识符
// 即使实体本身尚未在世界中实例化
// 这有效是因为 Commands 会在实际生成实体之前通过原子计数器保留实体 ID
let alice = commands.spawn(Name::new("Alice")).id();
let bob = commands.spawn((Name::new("Bob"), Targeting(alice))).id();

// 使用 entity() 方法获取实体命令
commands.entity(alice).insert(Targeting(bob));
```

**源代码文件**：`bevy/examples/ecs/entity_disabling.rs`

**代码示例**：

```rust
fn disable_entities_on_click(
    click: On<Pointer<Click>>,
    valid_query: Query<&DisableOnClick>,
    mut commands: Commands,
) {
    if valid_query.contains(click.entity) {
        // 只需将 `Disabled` 组件添加到实体即可禁用它
        commands.entity(click.entity).insert(Disabled);
    }
}

fn reenable_entities_on_space(
    mut commands: Commands,
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

**源代码文件**：`bevy/examples/ecs/fallible_params.rs`

**代码示例**：

```rust
fn user_input(
    mut commands: Commands,
    enemies: Query<Entity, With<Enemy>>,
    keyboard_input: Res<ButtonInput<KeyCode>>,
    asset_server: Res<AssetServer>,
) {
    if keyboard_input.just_pressed(KeyCode::KeyR)
        && let Some(entity) = enemies.iter().next()
    {
        // 使用 despawn() 销毁实体
        commands.entity(entity).despawn();
    }
}
```

**关键要点**：
- 使用 `entity()` 方法获取实体命令
- 使用 `insert()` 方法插入组件
- 使用 `remove()` 方法移除组件
- 使用 `despawn()` 方法销毁实体
- 可以链式调用多个操作

**说明**：
`entity()` 方法返回一个 `EntityCommands`，可以用于操作实体。可以链式调用多个操作，如 `commands.entity(entity).insert(component).remove::<OtherComponent>()`。

### 管理资源

使用 `insert_resource()` 和 `init_resource()` 管理资源。

**源代码文件**：`bevy/examples/ecs/ecs_guide.rs`

**代码示例**：

```rust
fn startup_system(mut commands: Commands, mut game_state: ResMut<GameState>) {
    // 创建游戏规则资源
    commands.insert_resource(GameRules {
        max_rounds: 10,
        winning_score: 4,
        max_players: 4,
    });
}
```

**关键要点**：
- 使用 `insert_resource()` 插入资源
- 使用 `init_resource()` 初始化资源（需要实现 `Default` 或 `FromWorld` trait）
- 资源可以在系统运行时插入或初始化

**说明**：
Commands 可以用于管理资源。使用 `insert_resource()` 可以插入资源，使用 `init_resource()` 可以初始化资源。

## 进阶用法

### 注册和运行系统

使用 `register_system()` 和 `run_system()` 注册和运行一次性系统。

**源代码文件**：`bevy/examples/ecs/one_shot_systems.rs`

**代码示例**：

```rust
use bevy::{
    ecs::system::{RunSystemOnce, SystemId},
    prelude::*,
};

#[derive(Component)]
struct Callback(SystemId);

fn setup_with_commands(mut commands: Commands) {
    // 注册系统并获取系统 ID
    let system_id = commands.register_system(system_a);
    commands.spawn((Callback(system_id), A));
}

/// 如果实体也有 `Triggered` 组件，则运行与每个 `Callback` 组件关联的系统
fn evaluate_callbacks(query: Query<(Entity, &Callback), With<Triggered>>, mut commands: Commands) {
    for (entity, callback) in query.iter() {
        // 运行一次性系统
        commands.run_system(callback.0);
        commands.entity(entity).remove::<Triggered>();
    }
}
```

**关键要点**：
- 使用 `register_system()` 注册系统并获取系统 ID
- 使用 `run_system()` 运行一次性系统
- 一次性系统在触发时运行一次，而不是每次更新都运行
- 一次性系统可以用于推送式逻辑

**注意事项**：
- 一次性系统在触发时运行一次
- 一次性系统可以减少很少运行的系统开销
- 一次性系统可以提高调度灵活性

**最佳实践**：
- 对于很少运行的系统，使用一次性系统
- 对于事件驱动的逻辑，使用一次性系统
- 注意一次性系统的生命周期管理

### 触发事件

使用 `trigger()` 方法触发事件。

**源代码文件**：`bevy/examples/ecs/observers.rs`

**代码示例**：

```rust
/// [`EntityEvent`] 是一种专门的 [`Event`] 类型，可以针对特定实体
/// 除了在触发时运行正常的"顶级"观察者（针对任何爆炸的实体）外
/// 它还会运行针对该事件的特定实体的任何观察者
#[derive(EntityEvent)]
struct Explode {
    entity: Entity,
}

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_observer(
            |explode_mines: On<ExplodeMines>,
             mines: Query<&Mine>,
             index: Res<SpatialIndex>,
             mut commands: Commands| {
                for entity in index.get_nearby(explode_mines.pos) {
                    let mine = mines.get(entity).unwrap();
                    if mine.pos.distance(explode_mines.pos) < mine.size + explode_mines.radius {
                        // 触发事件
                        commands.trigger(Explode { entity });
                    }
                }
            },
        )
        .run();
}
```

**关键要点**：
- 使用 `trigger()` 方法触发事件
- 事件可以是普通事件或实体事件
- 实体事件可以针对特定实体
- 观察者会响应触发的事件

**注意事项**：
- 事件可以是普通事件或实体事件
- 实体事件可以针对特定实体
- 观察者会响应触发的事件

**最佳实践**：
- 对于事件驱动的逻辑，使用事件
- 对于需要针对特定实体的逻辑，使用实体事件
- 注意事件的性能影响

### 发送消息

使用 `write_message()` 方法发送消息。

**源代码文件**：`bevy/examples/ecs/message.rs`

**代码示例**：

```rust
/// 这是一个 [`Message`]，任何观察它的观察者都会在它被激活时运行
#[derive(Message)]
struct MyMessage;

fn send_message(mut commands: Commands) {
    // 发送消息
    commands.write_message(MyMessage);
}

fn receive_message(_message: On<MyMessage>) {
    info!("Message received!");
}
```

**关键要点**：
- 使用 `write_message()` 方法发送消息
- 消息需要实现 `Message` trait
- 观察者会响应发送的消息
- 消息可以用于系统间通信

**注意事项**：
- 消息需要实现 `Message` trait
- 观察者会响应发送的消息
- 消息可以用于系统间通信

**最佳实践**：
- 对于系统间通信，使用消息
- 对于事件驱动逻辑，使用消息
- 注意消息的性能影响

### 实体层次结构

使用 `with_children()` 和 `add_child()` 管理实体层次结构。

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
```

**关键要点**：
- 使用 `with_children()` 方法在创建实体时添加子实体
- 使用 `add_child()` 方法在实体创建后添加子实体
- 使用 `remove_child()` 方法移除子实体
- 销毁父实体会自动销毁所有子实体

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

Commands 在游戏开发中有广泛的应用：

1. **实体创建**：创建玩家、敌人、道具等游戏对象
2. **组件管理**：添加、移除、修改组件
3. **资源管理**：插入和初始化资源
4. **事件驱动**：触发事件和发送消息
5. **系统管理**：注册和运行一次性系统

### 常见问题

**问题 1**：Commands 何时执行？

**解决方案**：Commands 在系统执行完成后统一应用。这意味着在系统执行期间，Commands 的修改不会立即生效。

**问题 2**：如何获取刚创建的实体 ID？

**解决方案**：使用 `.id()` 方法在创建实体后立即获取实体 ID。即使实体本身尚未在世界中实例化，也可以获取 ID。

**问题 3**：Commands 和直接访问 World 有什么区别？

**解决方案**：
- Commands 是线程安全的，可以在并行系统中使用
- 直接访问 World 需要独占系统，会阻止并行执行
- Commands 将修改操作排队，在系统执行完成后统一应用

### 性能考虑

1. **命令缓冲**：Commands 将修改操作排队，批量处理，减少开销
2. **延迟执行**：Commands 的修改在系统执行完成后统一应用，提高性能
3. **批量操作**：使用 `spawn_batch()` 批量创建实体，比逐个创建更高效

## 相关资源

**相关源代码文件**：
- `bevy/examples/ecs/ecs_guide.rs` - ECS 完整指南示例（Commands 使用）
- `bevy/examples/ecs/one_shot_systems.rs` - 一次性系统示例（注册和运行系统）
- `bevy/examples/ecs/observers.rs` - 观察者模式示例（触发事件）
- `bevy/examples/ecs/hierarchy.rs` - 实体层次结构示例（管理层次结构）
- `bevy/examples/ecs/entity_disabling.rs` - 实体禁用示例（操作实体）

**官方文档链接**：
- [Bevy Commands 官方文档](https://docs.rs/bevy/latest/bevy/ecs/system/struct.Commands.html)
- [EntityCommands 文档](https://docs.rs/bevy/latest/bevy/ecs/system/struct.EntityCommands.html)

**进一步学习建议**：
- 学习实体（Entities），了解如何创建和操作实体
- 学习系统（Systems），了解如何在系统中使用 Commands
- 学习 ECS 进阶，了解 Commands 的高级功能

---

**索引**：[返回上级目录](/wiki/BevyBook/ECS/)


