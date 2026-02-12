# 系统（Systems）## 概述

**学习目标**：
- 理解系统的概念和作用
- 掌握如何定义和使用系统
- 了解系统参数类型
- 理解系统闭包、泛型系统、一次性系统等高级功能

**前置知识要求**：
- 核心编程框架（ECS）
- 组件（Components）
- 实体（Entities）
- Rust 基础语法


## 核心概念

### 什么是系统？

系统是在实体、组件和资源上运行逻辑的函数。系统是 ECS 中的逻辑单元，用于处理游戏逻辑。

**为什么使用系统？**

1. **逻辑分离**：将游戏逻辑分离为独立的系统
2. **并行执行**：系统可以并行执行，提高性能
3. **易于测试**：系统可以独立测试

### 系统的设计思想

系统采用数据导向的设计思想，将数据（组件）和逻辑（系统）分离。这种设计使得：

- 系统可以独立开发和测试
- 系统可以并行执行，提高性能
- 系统可以灵活组合，实现复杂的游戏逻辑

## 基础用法

### 定义系统

系统是普通 Rust 函数，用于处理实体、组件和资源。

**源代码文件**：`bevy/examples/ecs/ecs_guide.rs`

**代码示例**：

```rust
// 这是最简单的系统类型。它只是每次运行时打印 "This game is fun!"
fn print_message_system() {
    println!("This game is fun!");
}

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

// 这个系统更新所有具有 `Player`、`Score` 和 `PlayerStreak` 组件的实体的分数
fn score_system(mut query: Query<(&Player, &mut Score, &mut PlayerStreak)>) {
    for (player, mut score, mut streak) in &mut query {
        let scored_a_point = random::<bool>();
        if scored_a_point {
            // 不可变访问组件通过常规引用完成 - `player` 的类型是 `&Player`
            // 可变访问组件通过 `Mut<T>` 类型完成 - `score` 的类型是 `Mut<Score>`
            // `Mut<T>` 实现了 `Deref<T>`，所以可以使用标准字段更新语法
            score.value += 1;
            // 匹配枚举需要解引用
            *streak = match *streak {
                PlayerStreak::Hot(n) => PlayerStreak::Hot(n + 1),
                PlayerStreak::Cold(_) | PlayerStreak::None => PlayerStreak::Hot(1),
            };
            println!(
                "{} scored a point! Their score is: {} ({})",
                player.name, score.value, *streak
            );
        }
    }
}
```

**关键要点**：
- 系统是普通 Rust 函数
- 系统可以访问资源（`Res<T>`、`ResMut<T>`）
- 系统可以查询组件（`Query<T>`）
- 系统可以创建和修改实体（`Commands`）
- 系统在每次应用更新时运行

**说明**：
系统是 ECS 中的逻辑单元。系统通过查询访问组件，通过 `Res` 或 `ResMut` 访问资源。系统可以并行执行，提高性能。

### 系统参数类型

系统可以使用多种参数类型来访问不同的数据。

**源代码文件**：`bevy/examples/ecs/ecs_guide.rs`

**代码示例**：

```rust
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
- `Res<T>`：只读访问资源
- `ResMut<T>`：可变访问资源
- `Query<T>`：查询组件
- `Commands`：创建和修改实体
- 系统可以同时使用多种参数类型

**说明**：
系统参数必须实现 `SystemParam` trait。Bevy 提供了多种系统参数类型，如 `Res`、`ResMut`、`Query`、`Commands` 等。

## 进阶用法

### 系统闭包

系统可以是闭包，允许捕获外部变量。

**源代码文件**：`bevy/examples/ecs/system_closure.rs`

**代码示例**：

```rust
fn main() {
    // 创建一个简单的闭包
    let simple_closure = || {
        // 这是一个什么都不做的闭包
        info!("Hello from a simple closure!");
    };

    // 创建一个带有'input'值的闭包
    let complex_closure = |mut value: String| {
        move || {
            info!("Hello from a complex closure! {}", value);

            // 我们可以在闭包内修改值。这将在调用之间保存
            value = format!("{value} - updated");
        }
    };

    let outside_variable = "bar".to_string();

    App::new()
        .add_plugins(LogPlugin::default())
        // 我们可以使用闭包作为系统
        .add_systems(Update, simple_closure)
        // 或者我们可以使用更复杂的闭包，并传递参数来初始化 Local 变量
        .add_systems(Update, complex_closure("foo".into()))
        // 我们也可以内联闭包
        .add_systems(Update, || {
            info!("Hello from an inlined closure!");
        })
        // 或使用闭包外部的变量
        .add_systems(Update, move || {
            info!(
                "Hello from an inlined closure that captured the 'outside_variable'! {}",
                outside_variable
            );
            // 你可以使用 outside_variable 或此闭包内的任何其他变量
            // 它们的状态将被保存
        })
        .run();
}
```

**关键要点**：
- 系统可以是闭包
- 闭包可以捕获外部变量
- 使用 `move` 关键字移动变量到闭包中
- 闭包可以用于简单的系统逻辑

**注意事项**：
- 闭包捕获的变量会在系统调用之间保持状态
- 使用 `move` 关键字会移动变量到闭包中
- 注意闭包的生命周期和所有权

**最佳实践**：
- 对于简单的系统逻辑，使用闭包
- 对于复杂的系统逻辑，使用普通函数
- 注意闭包捕获的变量的生命周期

### 泛型系统

泛型系统允许在不同类型上重用逻辑。

**源代码文件**：`bevy/examples/ecs/generic_system.rs`

**代码示例**：

```rust
// 类型参数在函数名之后，但在普通参数之前
// 这里，`Component` trait 是 T 的 trait 约束，我们的泛型类型
fn cleanup_system<T: Component>(mut commands: Commands, query: Query<Entity, With<T>>) {
    for e in &query {
        commands.entity(e).despawn();
    }
}

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .init_state::<AppState>()
        .add_systems(Startup, setup_system)
        .add_systems(
            Update,
            (
                print_text_system,
                transition_to_in_game_system.run_if(in_state(AppState::MainMenu)),
            ),
        )
        // 清理系统
        // 使用 ::<T> (turbofish) 语法传递系统应该操作的类型
        .add_systems(OnExit(AppState::MainMenu), cleanup_system::<MenuClose>)
        .add_systems(OnExit(AppState::InGame), cleanup_system::<LevelUnload>)
        .run();
}
```

**关键要点**：
- 泛型系统允许在不同类型上重用逻辑
- 使用 `::<T>` (turbofish) 语法指定类型
- 泛型类型必须满足 trait 约束
- 泛型系统可以用于处理相关组件或资源

**注意事项**：
- 泛型系统需要为每个类型创建专门的副本
- 确保为每个类型都注册了系统
- 泛型系统可以结合用户定义的 trait 使用

**最佳实践**：
- 对于处理相关组件或资源的系统，使用泛型系统
- 结合用户定义的 trait 使用泛型系统
- 确保为每个类型都注册了系统

### 一次性系统

一次性系统在触发时运行一次，而不是每次更新都运行。

**源代码文件**：`bevy/examples/ecs/one_shot_systems.rs`

**代码示例**：

```rust
use bevy::{
    ecs::system::{RunSystemOnce, SystemId},
    prelude::*,
};

#[derive(Component)]
struct Callback(SystemId);

#[derive(Component)]
struct Triggered;

fn setup_with_commands(mut commands: Commands) {
    let system_id = commands.register_system(system_a);
    commands.spawn((Callback(system_id), A));
}

fn setup_with_world(world: &mut World) {
    // 我们可以手动运行一次
    world.run_system_once(system_b).unwrap();
    // 或使用 Callback
    let system_id = world.register_system(system_b);
    world.spawn((Callback(system_id), B));
}

/// 用 `Triggered` 组件标记具有我们想要运行的回调的实体
fn trigger_system(
    mut commands: Commands,
    query_a: Single<Entity, With<A>>,
    query_b: Single<Entity, With<B>>,
    input: Res<ButtonInput<KeyCode>>,
) {
    if input.just_pressed(KeyCode::KeyA) {
        let entity = *query_a;
        commands.entity(entity).insert(Triggered);
    }
    if input.just_pressed(KeyCode::KeyB) {
        let entity = *query_b;
        commands.entity(entity).insert(Triggered);
    }
}

/// 如果实体也有 `Triggered` 组件，则运行与每个 `Callback` 组件关联的系统
///
/// 这可以在独占系统中完成，而不是使用 `Commands`（如果首选）
fn evaluate_callbacks(query: Query<(Entity, &Callback), With<Triggered>>, mut commands: Commands) {
    for (entity, callback) in query.iter() {
        commands.run_system(callback.0);
        commands.entity(entity).remove::<Triggered>();
    }
}

fn system_a(entity_a: Single<Entity, With<Text>>, mut writer: TextUiWriter) {
    *writer.text(*entity_a, 3) = String::from("A");
    info!("A: One shot system registered with Commands was triggered");
}

fn system_b(entity_b: Single<Entity, With<Text>>, mut writer: TextUiWriter) {
    *writer.text(*entity_b, 3) = String::from("B");
    info!("B: One shot system registered with World was triggered");
}
```

**关键要点**：
- 一次性系统在触发时运行一次
- 使用 `register_system()` 注册系统
- 使用 `run_system()` 或 `run_system_once()` 运行系统
- 一次性系统可以用于推送式逻辑

**注意事项**：
- 一次性系统可以减少很少运行的系统开销
- 一次性系统可以提高调度灵活性
- 一次性系统可以用于事件驱动的逻辑

**最佳实践**：
- 对于很少运行的系统，使用一次性系统
- 对于事件驱动的逻辑，使用一次性系统
- 注意一次性系统的生命周期管理

### 系统管道

系统管道允许将多个系统连接在一起，将第一个系统的输出传递给下一个系统。

**源代码文件**：`bevy/examples/ecs/system_piping.rs`

**代码示例**：

```rust
// 这个系统通过尝试解析 Message 资源产生 Result<usize> 输出
fn parse_message_system(message: Res<Message>) -> Result<usize, ParseIntError> {
    message.parse::<usize>()
}

// 这个系统接受 Result<usize> 输入，并打印解析的值或错误消息
// 尝试将 Message 资源更改为不是整数的内容。你应该看到错误消息被打印
fn handler_system(In(result): In<Result<usize, ParseIntError>>) {
    match result {
        Ok(value) => println!("parsed message: {value}"),
        Err(err) => println!("encountered an error: {err:?}"),
    }
}

fn main() {
    App::new()
        .insert_resource(Message("42".to_string()))
        .add_systems(
            Update,
            (
                parse_message_system.pipe(handler_system),
                data_pipe_system.map(|out| info!("{out}")),
                parse_message_system.map(|out| debug!("{out:?}")),
            ),
        )
        .run();
}
```

**关键要点**：
- 使用 `.pipe()` 方法将系统连接在一起
- 使用 `.map()` 方法转换系统输出
- 系统管道允许将多个系统连接在一起
- 系统管道可以用于错误处理和数据转换

**注意事项**：
- 系统管道要求系统输出类型匹配下一个系统的输入类型
- 系统管道可以用于错误处理
- 系统管道可以用于数据转换

**最佳实践**：
- 对于需要错误处理的系统，使用系统管道
- 对于需要数据转换的系统，使用系统管道
- 注意系统管道的类型匹配

### 独占系统

独占系统提供对 ECS World 的完全直接访问。它们不能与其他系统并行运行，因为它们可以访问任何东西并做任何事情。

**源代码文件**：`bevy/examples/ecs/ecs_guide.rs`

**代码示例**：

```rust
// 如果你真的需要完整、立即的读写访问世界或资源，你可以使用"独占系统"
// 警告：这些会阻止所有其他系统的并行执行，直到它们完成
// 所以如果你想要最大化并行性，通常应该避免它们
fn exclusive_player_system(world: &mut World) {
    // 这与 "new_player_system" 做同样的事情
    let total_players = world.resource_mut::<GameState>().total_players;
    let should_add_player = {
        let game_rules = world.resource::<GameRules>();
        let add_new_player = random::<bool>();
        add_new_player && total_players < game_rules.max_players
    };
    // 随机添加一个新玩家
    if should_add_player {
        println!("Player {} has joined the game!", total_players + 1);
        world.spawn((
            Player {
                name: format!("Player {}", total_players + 1),
            },
            Score { value: 0 },
            PlayerStreak::None,
        ));

        let mut game_state = world.resource_mut::<GameState>();
        game_state.total_players += 1;
    }
}
```

**关键要点**：
- 独占系统提供对 World 的完全访问
- 独占系统不能与其他系统并行运行
- 独占系统应该尽量避免，以最大化并行性
- 独占系统可以用于需要完全访问的场景

**注意事项**：
- 独占系统会阻止所有其他系统的并行执行
- 独占系统应该尽量避免
- 独占系统可以用于需要完全访问的场景

**最佳实践**：
- 尽量避免使用独占系统
- 只在必要时使用独占系统
- 考虑使用 `Commands` 代替独占系统

### 可失败系统参数

可失败系统参数在无法获取时会跳过系统，而不是导致错误。

**源代码文件**：`bevy/examples/ecs/fallible_params.rs`

**代码示例**：

```rust
// 这个系统只在恰好有一个匹配实体时运行
fn track_targets(
    // `Single` 确保系统只在恰好有一个匹配实体时运行
    mut player: Single<(&mut Transform, &Player)>,
    // `Option<Single>` 永远不会阻止系统运行，但如果没有恰好一个匹配实体，它将是 `None`
    enemy: Option<Single<&Transform, (With<Enemy>, Without<Player>)>>,
    time: Res<Time>,
) {
    let (player_transform, player) = &mut *player;
    if let Some(enemy_transform) = enemy {
        // 找到敌人，旋转并朝它移动
        // ...
    } else {
        // 找到 0 个或多个敌人，继续搜索
        // ...
    }
}

// 这个系统只在至少有一个匹配实体时运行
fn move_targets(mut enemies: Populated<(&mut Transform, &mut Enemy)>, time: Res<Time>) {
    for (mut transform, mut target) in &mut *enemies {
        // ...
    }
}
```

**关键要点**：
- `Single<D, F>`：必须有恰好一个匹配实体，否则系统会被静默跳过
- `Option<Single<D, F>>`：必须有零个或一个匹配实体，如果有多个，系统会被静默跳过
- `Populated<D, F>`：必须至少有一个匹配实体，否则系统会被静默跳过
- 可失败系统参数可以用于条件系统执行

**注意事项**：
- 可失败系统参数在无法获取时会跳过系统
- 其他系统参数（如 `Query`）永远不会失败验证
- 可失败系统参数可以用于条件系统执行

**最佳实践**：
- 对于需要恰好一个实体的系统，使用 `Single`
- 对于需要零个或一个实体的系统，使用 `Option<Single>`
- 对于需要至少一个实体的系统，使用 `Populated`

### 系统错误处理

系统可以返回 `Result` 来处理错误。

**源代码文件**：`bevy/examples/ecs/error_handling.rs`

**代码示例**：

```rust
use bevy::ecs::error::warn;

fn main() {
    let mut app = App::new();
    // 默认情况下，返回错误的可失败系统会 panic
    //
    // 我们可以通过设置自定义错误处理器来改变这一点，它适用于整个应用
    // （你也可以为特定的 `World` 设置它）
    // 这里我们使用内置错误处理器之一
    // Bevy 为 `panic`、`error`、`warn`、`info`、`debug`、`trace` 和 `ignore` 提供内置处理器
    app.set_error_handler(warn);

    app.add_plugins(DefaultPlugins);
    app.add_systems(Startup, setup);
    app.add_systems(Startup, failing_commands);

    // 单个系统也可以通过管道输出结果来处理：
    app.add_systems(
        PostStartup,
        failing_system.pipe(|result: In<Result>| {
            let _ = result.0.inspect_err(|err| info!("captured error: {err}"));
        }),
    );

    // 可失败观察者也被支持
    app.add_observer(fallible_observer);

    app.run();
}

/// 一个调用多个可失败函数并使用问号运算符的系统的示例
fn setup(
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<StandardMaterial>>,
) -> Result {
    // ...
    Ok(())
}

/// 这个系统总是失败验证，因为我们从未创建同时具有 `Player` 和 `Enemy` 组件的实体
fn failing_system(world: &mut World) -> Result {
    world
        // `get_resource` 返回 `Option<T>`，所以我们使用 `ok_or` 将其转换为 `Result`
        // 然后我们可以调用 `?` 来传播错误
        .get_resource::<UninitializedResource>()
        // 我们可以在这里提供 `str`，因为 `BevyError` 实现了 `From<&str>`
        .ok_or("Resource not initialized")?;

    Ok(())
}
```

**关键要点**：
- 系统可以返回 `Result<(), BevyError>` 来处理错误
- 使用 `set_error_handler()` 设置错误处理器
- 使用 `.pipe()` 方法处理系统错误
- 可失败系统可以用于错误处理

**注意事项**：
- 默认情况下，返回错误的系统会 panic
- 可以设置自定义错误处理器
- 系统错误可以通过管道处理

**最佳实践**：
- 对于可能失败的操作，使用可失败系统
- 设置适当的错误处理器
- 使用系统管道处理错误

## 实际应用

### 在游戏开发中的应用场景

系统在游戏开发中有广泛的应用：

1. **游戏逻辑**：移动系统、伤害系统、AI 系统等
2. **渲染系统**：渲染系统、UI 系统等
3. **物理系统**：物理模拟、碰撞检测等

### 常见问题

**问题 1**：何时使用普通系统，何时使用独占系统？

**解决方案**：
- 大多数情况下使用普通系统
- 只在需要完全访问 World 时使用独占系统
- 考虑使用 `Commands` 代替独占系统

**问题 2**：如何控制系统的执行顺序？

**解决方案**：
- 使用 `.before()` 和 `.after()` 方法控制顺序
- 使用系统集（SystemSet）组织系统
- 使用系统管道连接系统

**问题 3**：如何处理系统错误？

**解决方案**：
- 使用可失败系统返回 `Result`
- 设置适当的错误处理器
- 使用系统管道处理错误

### 性能考虑

1. **系统粒度**：保持系统粒度细，只访问需要的数据
2. **并行执行**：系统可以并行执行，但可变访问会阻止并行
3. **系统数量**：避免过多的小系统，考虑合并相关系统

## 相关资源

**相关源代码文件**：
- `bevy/examples/ecs/ecs_guide.rs` - ECS 完整指南示例（系统定义）
- `bevy/examples/ecs/system_closure.rs` - 系统闭包示例
- `bevy/examples/ecs/generic_system.rs` - 泛型系统示例
- `bevy/examples/ecs/one_shot_systems.rs` - 一次性系统示例
- `bevy/examples/ecs/system_piping.rs` - 系统管道示例
- `bevy/examples/ecs/system_param.rs` - 自定义系统参数示例
- `bevy/examples/ecs/error_handling.rs` - 系统错误处理示例
- `bevy/examples/ecs/fallible_params.rs` - 可失败系统参数示例

**官方文档链接**：
- [Bevy System 官方文档](https://docs.rs/bevy_ecs/latest/bevy_ecs/system/index.html)
- [SystemParam 文档](https://docs.rs/bevy_ecs/latest/bevy_ecs/system/trait.SystemParam.html)

**进一步学习建议**：
- 学习查询（Queries），了解如何访问组件
- 学习资源（Resources），了解如何访问资源
- 学习系统调度（Schedule & App），了解如何控制系统执行

---

**索引**：[返回上级目录](/wiki/bevybook/ecs/)


