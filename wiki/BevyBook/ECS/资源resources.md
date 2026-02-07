# 资源（Resources）## 概述

**学习目标**：
- 理解资源的概念和作用
- 掌握如何定义和使用资源
- 了解资源的初始化方法
- 理解资源的生命周期管理

**前置知识要求**：
- 核心编程框架（ECS）
- 组件（Components）
- 系统（Systems）
- Rust 基础语法


## 核心概念

### 什么是资源？

资源是共享的全局数据，可以在任何系统中访问。资源是全局唯一的，不需要附加到实体上。

**为什么使用资源？**

1. **全局状态**：资源用于存储全局状态，如游戏设置、配置信息等
2. **共享数据**：资源可以在任何系统中访问，便于共享数据
3. **性能优化**：资源访问比查询更高效

### 资源的设计思想

资源采用数据导向的设计思想，将全局数据（资源）和逻辑（系统）分离。这种设计使得：

- 系统可以高效访问全局数据
- 资源可以在任何系统中访问
- 资源可以灵活管理全局状态

## 基础用法

### 定义资源

资源是普通 Rust 数据类型，通过 `#[derive(Resource)]` 标记。

**源代码文件**：`bevy/examples/ecs/ecs_guide.rs`

**代码示例**：

```rust
// 这个资源保存游戏信息
#[derive(Resource, Default)]
struct GameState {
    current_round: usize,
    total_players: usize,
    winning_player: Option<String>,
}

// 这个资源提供游戏的规则
#[derive(Resource)]
struct GameRules {
    winning_score: usize,
    max_rounds: usize,
    max_players: usize,
}
```

**关键要点**：
- 资源是普通 Rust 结构体或枚举
- 使用 `#[derive(Resource)]` 标记资源
- 使用 `Default` trait 可以自动初始化资源
- 资源用于存储全局状态

**说明**：
资源用于存储全局状态，如游戏设置、配置信息等。与组件不同，资源是全局唯一的，不需要附加到实体上。

### 访问资源

使用 `Res<T>` 和 `ResMut<T>` 访问资源。

**源代码文件**：`bevy/examples/ecs/ecs_guide.rs`

**代码示例**：

```rust
// 系统也可以读取和修改资源。这个系统在每次更新时开始一个新的"轮次"
// 注意："mut" 表示资源是"可变的"
// Res<GameRules> 是只读的。ResMut<GameState> 可以修改资源
fn new_round_system(game_rules: Res<GameRules>, mut game_state: ResMut<GameState>) {
    game_state.current_round += 1;
    println!(
        "Begin round {} of {}",
        game_state.current_round, game_rules.max_rounds
    );
}

// 这个系统在所有具有 `Player` 和 `Score` 组件的实体上运行
// 它还访问 `GameRules` 资源来确定玩家是否获胜
fn score_check_system(
    game_rules: Res<GameRules>,
    mut game_state: ResMut<GameState>,
    query: Query<(&Player, &Score)>,
) {
    for (player, score) in &query {
        if score.value == game_rules.winning_score {
            game_state.winning_player = Some(player.name.clone());
        }
    }
}
```

**关键要点**：
- 使用 `Res<T>` 只读访问资源
- 使用 `ResMut<T>` 可变访问资源
- 资源可以在任何系统中访问
- 可变访问会阻止并行执行

**说明**：
资源访问与组件访问类似。使用 `Res<T>` 可以只读访问资源，使用 `ResMut<T>` 可以可变访问资源。资源可以在任何系统中访问，便于共享数据。

## 进阶用法

### 资源初始化

资源可以通过多种方式初始化。

**源代码文件**：`bevy/examples/ecs/ecs_guide.rs`

**代码示例**：

```rust
fn main() {
    App::new()
        // 实现 Default 或 FromWorld trait 的资源可以这样添加：
        .init_resource::<GameState>()
        // 或者使用 insert_resource 插入资源
        .insert_resource(GameRules {
            max_rounds: 10,
            winning_score: 4,
            max_players: 4,
        })
        .add_systems(Startup, startup_system)
        .add_systems(Update, new_round_system)
        .run();
}

fn startup_system(mut commands: Commands, mut game_state: ResMut<GameState>) {
    // 创建游戏规则资源
    commands.insert_resource(GameRules {
        max_rounds: 10,
        winning_score: 4,
        max_players: 4,
    });

    // 设置总玩家数为 2
    game_state.total_players = 2;
}
```

**关键要点**：
- 使用 `init_resource::<T>()` 初始化实现 `Default` 或 `FromWorld` trait 的资源
- 使用 `insert_resource()` 插入资源
- 使用 `Commands::insert_resource()` 在系统中插入资源
- 资源可以在应用构建时或系统运行时初始化

**注意事项**：
- `init_resource()` 要求资源实现 `Default` 或 `FromWorld` trait
- `insert_resource()` 可以插入任何资源
- 资源可以在应用构建时或系统运行时初始化

**最佳实践**：
- 对于有默认值的资源，使用 `init_resource()`
- 对于需要自定义初始化的资源，使用 `insert_resource()`
- 在启动系统中初始化资源

### 资源变更检测

资源支持变更检测，可以检测资源的变化。

**源代码文件**：`bevy/examples/ecs/change_detection.rs`

**代码示例**：

```rust
/// 资源的变更检测概念与组件类似
fn change_resource(time: Res<Time>, mut my_resource: ResMut<MyResource>) {
    if rand::rng().random_bool(0.1) {
        let new_resource = MyResource(time.elapsed_secs().round());
        info!("New value: {new_resource:?}");
        // 为了避免在值未实际改变时触发变更检测，可以使用 `set_if_neq` 方法
        // 该方法要求资源实现 PartialEq
        my_resource.set_if_neq(new_resource);
    }
}

fn change_detection(
    changed_components: Query<Ref<MyComponent>, Changed<MyComponent>>,
    my_resource: Res<MyResource>,
) {
    // ...

    if my_resource.is_changed() {
        warn!(
            "Change detected!\n\t-> value: {:?}\n\t-> added: {}\n\t-> changed: {}\n\t-> changed by: {}",
            my_resource,
            my_resource.is_added(),
            my_resource.is_changed(),
            my_resource.changed_by() // 与组件一样，需要 `track_location` 特性
        );
    }
}
```

**关键要点**：
- 使用 `is_changed()` 检查资源是否已变更
- 使用 `is_added()` 检查资源是否新添加
- 使用 `set_if_neq()` 避免不必要的变更检测
- 使用 `changed_by()` 获取变更位置（需要 `track_location` 特性）

**注意事项**：
- 资源的变更检测与组件类似
- 使用 `set_if_neq()` 避免不必要的变更检测
- `changed_by()` 需要 `track_location` 特性

**最佳实践**：
- 对于实现 `PartialEq` 的资源，使用 `set_if_neq()` 避免不必要的变更检测
- 使用变更检测响应资源变化
- 注意变更检测的性能影响

## 实际应用

### 在游戏开发中的应用场景

资源在游戏开发中有广泛的应用：

1. **游戏设置**：游戏配置、难度设置等
2. **全局状态**：游戏状态、分数、时间等
3. **资源管理**：资源服务器、资源缓存等

### 常见问题

**问题 1**：何时使用资源，何时使用组件？

**解决方案**：
- 资源用于全局共享的数据（如游戏设置、资源服务器）
- 组件用于实体特有的数据（如位置、速度）
- 如果数据是全局唯一的，使用资源；如果是实体特有的，使用组件

**问题 2**：如何初始化资源？

**解决方案**：
- 对于有默认值的资源，使用 `init_resource()`
- 对于需要自定义初始化的资源，使用 `insert_resource()`
- 在启动系统中初始化资源

**问题 3**：如何检测资源变化？

**解决方案**：
- 使用 `is_changed()` 检查资源是否已变更
- 使用 `is_added()` 检查资源是否新添加
- 使用 `set_if_neq()` 避免不必要的变更检测

### 性能考虑

1. **资源访问**：资源访问比查询更高效
2. **变更检测**：注意变更检测的性能影响
3. **资源数量**：避免过多的资源，考虑合并相关资源

## 相关资源

**相关源代码文件**：
- `bevy/examples/ecs/ecs_guide.rs` - ECS 完整指南示例（资源定义和使用）
- `bevy/examples/ecs/change_detection.rs` - 变更检测示例（资源变更检测）

**官方文档链接**：
- [Bevy Resource 官方文档](https://docs.rs/bevy_ecs/latest/bevy_ecs/system/struct.Res.html)
- [Resource 初始化文档](https://docs.rs/bevy/latest/bevy/app/struct.App.html#method.init_resource)

**进一步学习建议**：
- 学习系统（Systems），了解如何在系统中访问资源
- 学习系统调度（Schedule & App），了解资源的生命周期
- 学习 ECS 进阶，了解资源的高级功能

---

**索引**：[返回上级目录](/wiki/BevyBook/ECS/)


