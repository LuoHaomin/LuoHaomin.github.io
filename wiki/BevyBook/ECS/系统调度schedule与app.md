# 系统调度（Schedule & App）## 概述

**学习目标**：
- 理解 Schedule 的概念和作用
- 掌握如何控制系统执行顺序
- 了解自定义 Schedule 的创建方法
- 理解 App 的生命周期和运行条件

**前置知识要求**：
- 核心编程框架（ECS）
- 系统（Systems）
- 资源（Resources）
- Rust 基础语法


## 核心概念

### 什么是 Schedule？

Schedule 控制系统执行策略和每个 tick 内系统的广泛顺序。每个系统都属于一个 Schedule，Schedule 控制系统的执行顺序。

**为什么使用 Schedule？**

1. **执行顺序**：Schedule 控制系统的执行顺序
2. **执行策略**：Schedule 控制系统的执行策略（并行或顺序）
3. **生命周期**：Schedule 控制系统的生命周期（启动、更新、结束等）

### Schedule 的设计思想

Schedule 采用数据导向的设计思想，将系统执行顺序和逻辑分离。这种设计使得：

- 系统可以灵活组织
- 系统执行顺序可以明确控制
- 系统可以并行执行，提高性能

## 基础用法

### 默认 Schedule

Bevy 提供了多个默认 Schedule，如 `Startup`、`Update`、`Last` 等。

**源代码文件**：`bevy/examples/ecs/ecs_guide.rs`

**代码示例**：

```rust
fn main() {
    App::new()
        // `Startup` 系统在应用启动时恰好运行一次
        // 这些通常用于应用初始化代码（例如：添加实体和资源）
        .add_systems(Startup, startup_system)
        // `Update` 系统每次更新运行一次
        // 这些通常用于"实时应用逻辑"
        .add_systems(Update, print_message_system)
        // 还有其他 Schedule，如 `Last`，它在每次运行的末尾运行
        .add_systems(Last, print_at_end_round)
        .run();
}
```

**关键要点**：
- `Startup`：启动系统，在应用启动时运行一次
- `Update`：更新系统，每次更新运行一次
- `Last`：最后系统，在每次运行的末尾运行
- 系统按 Schedule 顺序执行

**说明**：
Bevy 提供了多个默认 Schedule，用于控制系统的执行顺序。`Startup` 系统在应用启动时运行一次，用于初始化。`Update` 系统每次更新运行一次，用于游戏逻辑。`Last` 系统在每次运行的末尾运行，用于清理。

### 系统执行顺序

可以通过 `.before()` 和 `.after()` 方法控制系统的执行顺序。

**源代码文件**：`bevy/examples/ecs/ecs_guide.rs`

**代码示例**：

```rust
// 系统执行顺序
//
// 每个系统属于一个 `Schedule`，它控制执行策略和每个 tick 内系统的广泛顺序
// `Startup` schedule 保存启动系统，它们在 `Update` 运行之前运行一次
// `Update` 每次应用更新运行一次，通常是一"帧"或一"tick"
//
// 默认情况下，`Schedule` 中的所有系统并行运行，除非它们需要对数据的可变访问
// 这是高效的，但有时顺序很重要
// 例如，我们希望我们的"游戏结束"系统在所有其他系统之后执行
// 以确保我们不会意外地多运行一轮游戏
//
// 你可以使用 `.before` 或 `.after` 方法强制系统之间的显式顺序
// 系统不会调度，直到它们具有"顺序依赖"的所有系统都已完成
// 还有其他 Schedule，如 `Last`，它在每次运行的末尾运行
.add_systems(Last, print_at_end_round)
```

**关键要点**：
- 默认情况下，系统并行运行
- 使用 `.before()` 和 `.after()` 方法控制顺序
- 系统不会调度，直到依赖的系统完成
- 可变访问会阻止并行执行

**说明**：
默认情况下，系统并行运行，除非它们需要对数据的可变访问。使用 `.before()` 和 `.after()` 方法可以明确控制系统的执行顺序。系统不会调度，直到它们依赖的所有系统都已完成。

### 系统集（SystemSet）

系统集用于组织相关系统，并控制它们的执行顺序。

**源代码文件**：`bevy/examples/ecs/ecs_guide.rs`

**代码示例**：

```rust
/// 一组相关的系统集，用于控制系统顺序
/// 系统可以添加到任意数量的集合中
#[derive(SystemSet, Debug, Hash, PartialEq, Eq, Clone)]
enum MySystems {
    BeforeRound,
    Round,
    AfterRound,
}

fn main() {
    App::new()
        // 我们也可以创建新的系统集，并相对于其他系统集对它们进行排序
        // 这是我们游戏的执行顺序：
        // "before_round": new_player_system, new_round_system
        // "round": print_message_system, score_system
        // "after_round": score_check_system, game_over_system
        .configure_sets(
            Update,
            // chain() 将确保集合按列出的顺序运行
            (
                MySystems::BeforeRound,
                MySystems::Round,
                MySystems::AfterRound,
            )
                .chain(),
        )
        // add_systems 函数很强大。你可以轻松定义复杂的系统配置！
        .add_systems(
            Update,
            (
                // 这些 `BeforeRound` 系统将在 `Round` 系统之前运行，这要归功于链式集合配置
                (
                    // 你也可以链式系统！new_round_system 将首先运行，然后是 new_player_system
                    (new_round_system, new_player_system).chain(),
                    exclusive_player_system,
                )
                    // 上面元组中的所有系统都将添加到这个集合中
                    .in_set(MySystems::BeforeRound),
                // 这个 `Round` 系统将在 `BeforeRound` 系统之后运行，这要归功于链式集合配置
                score_system.in_set(MySystems::Round),
                // 这些 `AfterRound` 系统将在 `Round` 系统之后运行，这要归功于链式集合配置
                (
                    score_check_system,
                    // 除了 chain()，你还可以使用 `before(system)` 和 `after(system)`
                    // 这也适用于集合！
                    game_over_system.after(score_check_system),
                )
                    .in_set(MySystems::AfterRound),
            ),
        )
        .run();
}
```

**关键要点**：
- 使用 `SystemSet` 组织相关系统
- 使用 `configure_sets()` 配置系统集的顺序
- 使用 `.chain()` 方法链式系统集
- 使用 `.in_set()` 方法将系统添加到系统集

**说明**：
系统集用于组织相关系统，并控制它们的执行顺序。使用 `configure_sets()` 可以配置系统集的顺序，使用 `.chain()` 方法可以链式系统集。

## 进阶用法

### 自定义 Schedule

可以创建自定义 Schedule，用于特定的执行策略。

**源代码文件**：`bevy/examples/ecs/custom_schedule.rs`

**代码示例**：

```rust
use bevy::{
    app::MainScheduleOrder,
    ecs::schedule::{ExecutorKind, ScheduleLabel},
    prelude::*,
};

#[derive(ScheduleLabel, Debug, Hash, PartialEq, Eq, Clone)]
struct SingleThreadedUpdate;

#[derive(ScheduleLabel, Debug, Hash, PartialEq, Eq, Clone)]
struct CustomStartup;

fn main() {
    let mut app = App::new();

    // 创建一个新的 [`Schedule`]
    // 为了演示目的，我们将其配置为使用单线程执行器
    // 这样此 Schedule 中的系统永远不会并行运行
    // 但是，这不是自定义 Schedule 的一般要求
    let mut custom_update_schedule = Schedule::new(SingleThreadedUpdate);
    custom_update_schedule.set_executor_kind(ExecutorKind::SingleThreaded);

    // 将 Schedule 添加到应用不会自动运行它
    // 这只是注册 Schedule，以便系统可以使用 `Schedules` 资源查找它
    app.add_schedule(custom_update_schedule);

    // Bevy `App` 有一个 `main_schedule_label` 字段，它配置应用运行器运行哪个 Schedule
    // 默认情况下，这是 `Main`
    // `Main` Schedule 负责运行 Bevy 的主要 Schedule，如 `Update`、`Startup` 或 `Last`
    //
    // 我们可以通过修改 `MainScheduleOrder` 资源来配置 `Main` Schedule 运行我们的自定义更新 Schedule
    // 相对于现有的 Schedule
    //
    // 注意，我们在 `main` 中直接修改 `MainScheduleOrder`，而不是在启动系统中
    // 原因是 `MainScheduleOrder` 不能从作为 `Main` Schedule 一部分运行的系统修改
    let mut main_schedule_order = app.world_mut().resource_mut::<MainScheduleOrder>();
    main_schedule_order.insert_after(Update, SingleThreadedUpdate);

    // 添加自定义启动 Schedule 的工作方式类似，但需要使用 `insert_startup_after`
    // 而不是 `insert_after`
    app.add_schedule(Schedule::new(CustomStartup));

    let mut main_schedule_order = app.world_mut().resource_mut::<MainScheduleOrder>();
    main_schedule_order.insert_startup_after(PreStartup, CustomStartup);

    app.add_systems(SingleThreadedUpdate, single_threaded_update_system)
        .add_systems(CustomStartup, custom_startup_system)
        .add_systems(PreStartup, pre_startup_system)
        .add_systems(Startup, startup_system)
        .add_systems(First, first_system)
        .add_systems(Update, update_system)
        .add_systems(Last, last_system)
        .run();
}
```

**关键要点**：
- 使用 `Schedule::new()` 创建自定义 Schedule
- 使用 `ScheduleLabel` 派生宏定义 Schedule 标签
- 使用 `set_executor_kind()` 设置执行器类型
- 使用 `MainScheduleOrder` 配置 Schedule 顺序

**注意事项**：
- 自定义 Schedule 需要添加到应用
- 需要配置 `MainScheduleOrder` 来运行自定义 Schedule
- 可以使用 `ExecutorKind::SingleThreaded` 创建单线程 Schedule

**最佳实践**：
- 对于需要特定执行策略的系统，使用自定义 Schedule
- 对于需要单线程执行的系统，使用单线程 Schedule
- 注意自定义 Schedule 的执行顺序

### 固定时间步（Fixed Timestep）

固定时间步允许系统以固定时间间隔运行，而不是每帧运行。

**源代码文件**：`bevy/examples/ecs/fixed_timestep.rs`

**代码示例**：

```rust
use bevy::prelude::*;

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        // 这个系统将每次更新运行一次（它应该匹配你的屏幕刷新率）
        .add_systems(Update, frame_update)
        // 将我们的系统添加到固定时间步 Schedule
        .add_systems(FixedUpdate, fixed_update)
        // 配置我们的固定时间步 Schedule 每秒运行两次
        .insert_resource(Time::<Fixed>::from_seconds(0.5))
        .run();
}

fn frame_update(mut last_time: Local<f32>, time: Res<Time>) {
    // 默认 `Time` 这里是 `Time<Virtual>`
    info!(
        "time since last frame_update: {}",
        time.elapsed_secs() - *last_time
    );
    *last_time = time.elapsed_secs();
}

fn fixed_update(mut last_time: Local<f32>, time: Res<Time>, fixed_time: Res<Time<Fixed>>) {
    // 默认 `Time` 这里是 `Time<Fixed>`
    info!(
        "time since last fixed_update: {}\n",
        time.elapsed_secs() - *last_time
    );

    info!("fixed timestep: {}\n", time.delta_secs());
    // 如果我们想看到超步，我们需要专门访问 `Time<Fixed>`
    info!(
        "time accrued toward next fixed_update: {}\n",
        fixed_time.overstep().as_secs_f32()
    );
    *last_time = time.elapsed_secs();
}
```

**关键要点**：
- 使用 `FixedUpdate` Schedule 运行固定时间步系统
- 使用 `Time::<Fixed>::from_seconds()` 配置固定时间步
- 固定时间步系统以固定时间间隔运行
- 固定时间步适合物理模拟等需要稳定时间步的场景

**注意事项**：
- 固定时间步系统以固定时间间隔运行
- 固定时间步适合物理模拟等需要稳定时间步的场景
- 注意固定时间步与帧率的区别

**最佳实践**：
- 对于物理模拟等需要稳定时间步的系统，使用固定时间步
- 对于与帧率相关的系统，使用 `Update` Schedule
- 注意固定时间步与帧率的区别

### 运行条件（Run Conditions）

运行条件允许控制系统是否应该运行。

**源代码文件**：`bevy/examples/ecs/run_conditions.rs`

**代码示例**：

```rust
use bevy::prelude::*;

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .init_resource::<InputCounter>()
        .add_systems(
            Update,
            (
                increment_input_counter
                    // common_conditions 模块有一些有用的运行条件
                    // 用于检查资源和状态
                    // 这些包含在 prelude 中
                    .run_if(resource_exists::<InputCounter>)
                    // `.or()` 是一个运行条件组合器，只有在第一个条件返回 `false` 时才评估第二个条件
                    // 这种行为称为"短路"，是 Rust 中 `||` 运算符的工作方式（以及大多数 C 系列语言）
                    // 在这种情况下，`has_user_input` 运行条件将被评估，因为 `Unused` 资源尚未初始化
                    .run_if(resource_exists::<Unused>.or(
                        // 这是一个自定义运行条件，使用返回 `bool` 的系统定义
                        // 并且具有只读 `SystemParam`
                        // 只有一个运行条件必须返回 `true`，系统才会运行
                        has_user_input,
                    )),
                print_input_counter
                    // `.and()` 是一个运行条件组合器，只有在第一个条件返回 `true` 时才评估第二个条件
                    // 类似于 `&&` 运算符
                    // 在这种情况下，短路行为防止第二个运行条件在 `InputCounter` 资源未初始化时 panic
                    .run_if(resource_exists::<InputCounter>.and(
                        // 这是一个闭包形式的自定义运行条件
                        // 这对于不需要重用的小型、简单的运行条件很有用
                        // 所有正常规则仍然适用：所有参数必须是只读的，除了本地参数
                        |counter: Res<InputCounter>| counter.is_changed() && !counter.is_added(),
                    )),
                print_time_message
                    // 这个函数返回一个自定义运行条件，很像 common_conditions 模块
                    // 它只会在 2 秒过去后返回 true
                    .run_if(time_passed(2.0))
                    // 你可以使用 common_conditions 模块中的 `not` 条件
                    // 来反转运行条件
                    // 在这种情况下，如果自应用启动以来经过的时间少于 2.5 秒，它将返回 true
                    .run_if(not(time_passed(2.5))),
            ),
        )
        .run();
}

/// 如果任何定义的输入刚刚被按下，则返回 true
///
/// 这是一个自定义运行条件，它可以接受任何正常的系统参数
/// 只要它们是只读的（除了本地参数可以是可变的）
/// 它返回一个 bool，决定系统是否应该运行
fn has_user_input(
    keyboard_input: Res<ButtonInput<KeyCode>>,
    mouse_button_input: Res<ButtonInput<MouseButton>>,
    touch_input: Res<Touches>,
) -> bool {
    keyboard_input.just_pressed(KeyCode::Space)
        || keyboard_input.just_pressed(KeyCode::Enter)
        || mouse_button_input.just_pressed(MouseButton::Left)
        || mouse_button_input.just_pressed(MouseButton::Right)
        || touch_input.any_just_pressed()
}

/// 这是一个返回闭包的函数，可以用作运行条件
///
/// 这很有用，因为你可以重用相同的运行条件，但使用不同的变量
/// 这就是 common_conditions 模块的工作方式
fn time_passed(t: f32) -> impl FnMut(Local<f32>, Res<Time>) -> bool {
    move |mut timer: Local<f32>, time: Res<Time>| {
        // 计时器计时
        *timer += time.delta_secs();
        // 如果计时器已经超过时间，返回 true
        *timer >= t
    }
}
```

**关键要点**：
- 使用 `.run_if()` 方法添加运行条件
- 使用 `.and()` 和 `.or()` 组合运行条件
- 使用 `not()` 反转运行条件
- 运行条件可以是函数或闭包

**注意事项**：
- 运行条件必须返回 `bool`
- 运行条件参数必须是只读的（除了本地参数）
- 只有所有运行条件都返回 `true` 时，系统才会运行

**最佳实践**：
- 对于条件系统执行，使用运行条件
- 使用 `.and()` 和 `.or()` 组合运行条件
- 注意运行条件的性能影响

### 系统步进（System Stepping）

系统步进允许逐步执行系统，用于调试。

**源代码文件**：`bevy/examples/ecs/system_stepping.rs`

**代码示例**：

```rust
use bevy::{ecs::schedule::Stepping, log::LogPlugin, prelude::*};

fn main() {
    let mut app = App::new();

    app.add_plugins(LogPlugin::default())
        .add_systems(
            Update,
            (
                update_system_one,
                // 在这里建立依赖关系以简化下面的描述
                update_system_two.after(update_system_one),
                update_system_three.after(update_system_two),
                update_system_four,
            ),
        )
        .add_systems(PreUpdate, pre_update_system);

    // 添加 Stepping 资源
    app.insert_resource(Stepping::new());
    
    // 将 Update Schedule 添加到 Stepping
    // 启用 Stepping
    let mut stepping = app.world_mut().resource_mut::<Stepping>();
    stepping.add_schedule(Update).enable();
    
    // 步进一帧
    stepping.step_frame();
    app.update();
    
    // 继续执行剩余系统
    stepping.continue_frame();
    app.update();
}
```

**关键要点**：
- 使用 `Stepping` 资源控制系统步进
- 使用 `step_frame()` 步进一帧
- 使用 `continue_frame()` 继续执行剩余系统
- 使用 `always_run()` 和 `never_run()` 控制特定系统的执行

**注意事项**：
- 系统步进需要启用 `bevy_debug_stepping` 特性
- 系统步进主要用于调试
- 注意系统步进对性能的影响

**最佳实践**：
- 对于调试，使用系统步进
- 对于生产环境，禁用系统步进
- 注意系统步进对性能的影响

### 非确定性系统顺序

默认情况下，Bevy 系统并行运行，除非明确指定顺序，否则它们的相对顺序是非确定性的。

**源代码文件**：`bevy/examples/ecs/nondeterministic_system_order.rs`

**代码示例**：

```rust
use bevy::{
    ecs::schedule::{LogLevel, ScheduleBuildSettings},
    prelude::*,
};

fn main() {
    App::new()
        // 我们可以按每个 Schedule 修改系统执行顺序歧义的报告策略
        // 你必须为每个要检查的 Schedule 执行此操作
        // 在已检查的 Schedule 内执行的子 Schedule 不会继承此修改
        .edit_schedule(Update, |schedule| {
            schedule.set_build_settings(ScheduleBuildSettings {
                ambiguity_detection: LogLevel::Warn,
                ..default()
            });
        })
        .init_resource::<A>()
        .init_resource::<B>()
        .add_systems(
            Update,
            (
                // 这对系统有歧义顺序
                // 因为它们的数据访问冲突，并且它们之间没有顺序
                reads_a,
                writes_a,
                // 这对系统有冲突的数据访问
                // 但通过显式顺序解决：
                // 这里的 .after 关系意味着我们总是在添加之后加倍
                adds_one_to_b,
                doubles_b.after(adds_one_to_b),
                // 这个系统与 adds_one_to_b 没有歧义
                // 由于我们的约束创建的传递顺序：
                // 如果 A 在 B 之前，B 在 C 之前，那么 A 必须在 C 之前
                reads_b.after(doubles_b),
                // 这个系统将与我们的所有写入系统冲突
                // 但我们已经用 adds_one_to_b 静默了它的歧义
                // 这应该只在明显的假阳性情况下完成：
                // 在你的代码中留下注释证明这个决定！
                reads_a_and_b.ambiguous_with(adds_one_to_b),
            ),
        )
        .add_plugins(DefaultPlugins)
        .run();
}
```

**关键要点**：
- 默认情况下，系统并行运行，顺序是非确定性的
- 使用 `.after()` 和 `.before()` 明确指定顺序
- 使用 `.ambiguous_with()` 静默歧义
- 使用 `ScheduleBuildSettings` 配置歧义检测

**注意事项**：
- 非确定性顺序可能导致微妙的错误
- 使用 `.after()` 和 `.before()` 明确指定顺序
- 注意歧义检测的性能影响

**最佳实践**：
- 对于有数据访问冲突的系统，明确指定顺序
- 使用 `.ambiguous_with()` 静默明显的假阳性
- 注意歧义检测的性能影响

## 实际应用

### 在游戏开发中的应用场景

系统调度在游戏开发中有广泛的应用：

1. **游戏逻辑**：使用 `Update` Schedule 运行游戏逻辑
2. **物理模拟**：使用 `FixedUpdate` Schedule 运行物理模拟
3. **初始化**：使用 `Startup` Schedule 运行初始化代码
4. **清理**：使用 `Last` Schedule 运行清理代码

### 常见问题

**问题 1**：如何控制系统执行顺序？

**解决方案**：
- 使用 `.before()` 和 `.after()` 方法控制顺序
- 使用系统集组织系统
- 使用 `.chain()` 方法链式系统

**问题 2**：如何创建自定义 Schedule？

**解决方案**：
- 使用 `Schedule::new()` 创建自定义 Schedule
- 使用 `ScheduleLabel` 派生宏定义 Schedule 标签
- 使用 `MainScheduleOrder` 配置 Schedule 顺序

**问题 3**：如何控制系统是否运行？

**解决方案**：
- 使用 `.run_if()` 方法添加运行条件
- 使用 `.and()` 和 `.or()` 组合运行条件
- 使用 `not()` 反转运行条件

### 性能考虑

1. **并行执行**：系统可以并行执行，但可变访问会阻止并行
2. **系统顺序**：明确指定系统顺序可以提高性能
3. **运行条件**：注意运行条件的性能影响

## 相关资源

**相关源代码文件**：
- `bevy/examples/ecs/ecs_guide.rs` - ECS 完整指南示例（系统调度）
- `bevy/examples/ecs/custom_schedule.rs` - 自定义 Schedule 示例
- `bevy/examples/ecs/fixed_timestep.rs` - 固定时间步示例
- `bevy/examples/ecs/run_conditions.rs` - 运行条件示例
- `bevy/examples/ecs/system_stepping.rs` - 系统步进示例
- `bevy/examples/ecs/nondeterministic_system_order.rs` - 非确定性系统顺序示例

**官方文档链接**：
- [Bevy Schedule 官方文档](https://docs.rs/bevy_ecs/latest/bevy_ecs/schedule/index.html)
- [App 官方文档](https://docs.rs/bevy/latest/bevy/app/struct.App.html)
- [SystemSet 文档](https://docs.rs/bevy_ecs/latest/bevy_ecs/schedule/trait.SystemSet.html)

**进一步学习建议**：
- 学习系统（Systems），了解如何定义系统
- 学习资源（Resources），了解如何访问资源
- 学习 ECS 进阶，了解系统调度的高级功能

---

**索引**：[返回上级目录](/wiki/bevybook/ecs/)


