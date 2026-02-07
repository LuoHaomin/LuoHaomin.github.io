# 时间系统（Time）## 概述

**学习目标**：
- 理解 Bevy 时间系统的基本概念
- 掌握时间资源的使用
- 了解定时器的使用
- 学会使用虚拟时间

**前置知识要求**：
- Bevy 快速入门
- ECS 基础
- 资源（Resources）基础


## 核心概念

### 什么是时间系统？

时间系统是 Bevy 中用于管理游戏时间的功能。时间系统提供了多种时间类型，包括真实时间、虚拟时间和固定时间步。

**为什么需要时间系统？**

1. **游戏逻辑**：时间系统可以控制游戏逻辑的执行速度
2. **动画**：时间系统可以控制动画的播放速度
3. **物理模拟**：时间系统可以控制物理模拟的时间步
4. **游戏暂停**：时间系统可以实现游戏暂停功能

### 时间系统的核心组件

Bevy 时间系统包含以下核心组件：

- **Time<Real>**：真实时间，不受游戏速度影响
- **Time<Virtual>**：虚拟时间，受游戏速度影响
- **Time<Fixed>**：固定时间步，用于物理模拟
- **Timer**：定时器，用于延迟和周期性任务

## 基础用法

### 时间资源

使用时间资源获取时间信息。

**源代码文件**：`bevy/examples/time/time.rs`

**代码示例**：

```rust
use bevy::{app::AppExit, prelude::*};
use std::{
    io::{self, BufRead},
    time::Duration,
};

fn main() {
    App::new()
        .add_plugins(MinimalPlugins)
        .insert_resource(Time::<Virtual>::from_max_delta(Duration::from_secs(5)))
        .insert_resource(Time::<Fixed>::from_duration(Duration::from_secs(1)))
        .add_systems(PreUpdate, print_real_time)
        .add_systems(FixedUpdate, print_fixed_time)
        .add_systems(Update, print_time)
        .set_runner(runner)
        .run();
}

fn print_real_time(time: Res<Time<Real>>) {
    println!(
        "PreUpdate: this is real time clock, delta is {:?} and elapsed is {:?}",
        time.delta(),
        time.elapsed()
    );
}

fn print_fixed_time(time: Res<Time>) {
    println!(
        "FixedUpdate: this is generic time clock inside fixed, delta is {:?} and elapsed is {:?}",
        time.delta(),
        time.elapsed()
    );
}

fn print_time(time: Res<Time>) {
    println!(
        "Update: this is generic time clock, delta is {:?} and elapsed is {:?}",
        time.delta(),
        time.elapsed()
    );
}
```

**关键要点**：
- 使用 `Time<Real>` 获取真实时间
- 使用 `Time<Virtual>` 获取虚拟时间
- 使用 `Time<Fixed>` 获取固定时间步
- 使用 `time.delta()` 获取时间增量
- 使用 `time.elapsed()` 获取经过的时间

**说明**：
时间资源是时间系统的基础。通过使用时间资源，可以获取时间信息，控制游戏逻辑的执行速度。

### 定时器

使用定时器实现延迟和周期性任务。

**源代码文件**：`bevy/examples/time/timers.rs`

**代码示例**：

```rust
use bevy::{log::info, prelude::*};

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .init_resource::<Countdown>()
        .add_systems(Startup, setup)
        .add_systems(Update, (countdown, print_when_completed))
        .run();
}

#[derive(Component, Deref, DerefMut)]
struct PrintOnCompletionTimer(Timer);

#[derive(Resource)]
struct Countdown {
    percent_trigger: Timer,
    main_timer: Timer,
}

impl Countdown {
    pub fn new() -> Self {
        Self {
            percent_trigger: Timer::from_seconds(4.0, TimerMode::Repeating),
            main_timer: Timer::from_seconds(20.0, TimerMode::Once),
        }
    }
}

impl Default for Countdown {
    fn default() -> Self::new()
    }
}

fn setup(mut commands: Commands) {
    // 向世界添加一个带有定时器的实体
    commands.spawn(PrintOnCompletionTimer(Timer::from_seconds(
        5.0,
        TimerMode::Once,
    )));
}

/// 此系统使用 bevy 的 `Time` 资源来获取每次更新之间的增量，
/// 对具有 `PrintOnCompletionTimer` 组件的实体上的 `Timer` 进行计时。
fn print_when_completed(time: Res<Time>, mut query: Query<&mut PrintOnCompletionTimer>) {
    for mut timer in &mut query {
        if timer.tick(time.delta()).just_finished() {
            info!("Entity timer just finished");
        }
    }
}

/// 此系统控制倒计时资源内的定时器计时并处理其状态。
fn countdown(time: Res<Time>, mut countdown: ResMut<Countdown>) {
    countdown.main_timer.tick(time.delta());

    // API 鼓励这种定时器状态检查（如果您只检查一个值）
    // 此外，由于定时器是重复的，`is_finished()` 会完成与 `just_finished` 相同的事情，
    // 但这在视觉上更有意义。
    if countdown.percent_trigger.tick(time.delta()).just_finished() {
        if !countdown.main_timer.is_finished() {
            // 打印主定时器完成的百分比。
            info!(
                "Timer is {:0.0}% complete!",
                countdown.main_timer.fraction() * 100.0
            );
        } else {
            // 定时器已完成，因此我们暂停百分比输出定时器
            countdown.percent_trigger.pause();
            info!("Paused percent trigger timer");
        }
    }
}
```

**关键要点**：
- 使用 `Timer::from_seconds()` 创建定时器
- 使用 `TimerMode::Once` 创建单次定时器
- 使用 `TimerMode::Repeating` 创建重复定时器
- 使用 `timer.tick()` 更新定时器
- 使用 `timer.just_finished()` 检查定时器是否刚完成
- 使用 `timer.is_finished()` 检查定时器是否完成
- 使用 `timer.fraction()` 获取定时器完成的百分比

**说明**：
定时器是时间系统的重要功能。通过使用定时器，可以实现延迟和周期性任务。

## 进阶用法

### 虚拟时间

使用虚拟时间控制游戏速度。

**源代码文件**：`bevy/examples/time/virtual_time.rs`

**代码示例**：

```rust
use std::time::Duration;
use bevy::{
    color::palettes::css::*, input::common_conditions::input_just_pressed, prelude::*,
    time::common_conditions::on_real_timer,
};

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Startup, setup)
        .add_systems(
            Update,
            (
                move_virtual_time_sprites,
                move_real_time_sprites,
                toggle_pause.run_if(input_just_pressed(KeyCode::Space)),
                change_time_speed::<1>.run_if(input_just_pressed(KeyCode::ArrowUp)),
                change_time_speed::<-1>.run_if(input_just_pressed(KeyCode::ArrowDown)),
                (update_virtual_time_info_text, update_real_time_info_text)
                    // 在定时器上更新文本以使其更易读
                    // `on_timer` 运行条件使用 `Virtual` 时间，这意味着它是缩放的
                    // 并且会根据 `Time<Virtual>::relative_speed` 和 `Time<Virtual>::is_paused()` 以不同的间隔更新 UI
                    .run_if(on_real_timer(Duration::from_millis(250))),
            ),
        )
        .run();
}

/// `Real` 时间相关标记
#[derive(Component)]
struct RealTime;

/// `Virtual` 时间相关标记
#[derive(Component)]
struct VirtualTime;

/// 设置示例
fn setup(mut commands: Commands, asset_server: Res<AssetServer>, mut time: ResMut<Time<Virtual>>) {
    // 从双倍 `Virtual` 时间开始，导致一个精灵以另一个精灵的两倍速度移动，
    // 另一个精灵基于 `Real`（未缩放）时间移动
    time.set_relative_speed(2.);

    commands.spawn(Camera2d);

    // ... 创建精灵和 UI ...
}

/// 使用 `Virtual`（缩放）时间移动精灵
fn move_virtual_time_sprites(
    mut sprite_query: Query<&mut Transform, (With<Sprite>, With<VirtualTime>)>,
    // 默认 `Time` 在常规系统中是 `Time<Virtual>`，
    // 或在固定时间步系统中是 `Time<Fixed>`，因此 `Time::delta()`、
    // `Time::elapsed()` 将返回适当的值
    time: Res<Time>,
) {
    for mut transform in sprite_query.iter_mut() {
        // 在 `Virtual` 秒内移动大约屏幕的一半
        // 当使用 `Time<Virtual>::set_relative_speed` 缩放时间时，
        // 它会以不同的速度移动，当时间暂停时精灵会保持静止
        transform.translation.x = get_sprite_translation_x(time.elapsed_secs());
    }
}

/// 更新 `Time<Virtual>` 的速度，增量为 `DELTA`
fn change_time_speed<const DELTA: i8>(mut time: ResMut<Time<Virtual>>) {
    let time_speed = (time.relative_speed() + DELTA as f32)
        .round()
        .clamp(0.25, 5.);

    // 设置虚拟时间的速度以加快或减慢速度
    time.set_relative_speed(time_speed);
}

/// 暂停或恢复 `Relative` 时间
fn toggle_pause(mut time: ResMut<Time<Virtual>>) {
    if time.is_paused() {
        time.unpause();
    } else {
        time.pause();
    }
}
```

**关键要点**：
- 使用 `Time<Virtual>` 获取虚拟时间
- 使用 `time.set_relative_speed()` 设置相对速度
- 使用 `time.pause()` 暂停时间
- 使用 `time.unpause()` 恢复时间
- 使用 `time.is_paused()` 检查时间是否暂停

**说明**：
虚拟时间是时间系统的高级功能。通过使用虚拟时间，可以控制游戏速度，实现游戏暂停和加速功能。

## 实际应用

### 在游戏开发中的应用场景

时间系统在游戏开发中有广泛的应用：

1. **游戏逻辑**：控制游戏逻辑的执行速度
2. **动画**：控制动画的播放速度
3. **物理模拟**：控制物理模拟的时间步
4. **游戏暂停**：实现游戏暂停功能
5. **定时任务**：实现延迟和周期性任务

### 常见问题

**问题 1**：如何获取时间增量？

**解决方案**：
- 使用 `Time::delta()` 获取时间增量
- 使用 `Time::delta_secs()` 获取时间增量（秒）
- 使用 `Time::delta_secs_f64()` 获取时间增量（秒，f64）

**问题 2**：如何创建定时器？

**解决方案**：
- 使用 `Timer::from_seconds()` 创建定时器
- 使用 `TimerMode::Once` 创建单次定时器
- 使用 `TimerMode::Repeating` 创建重复定时器

**问题 3**：如何控制游戏速度？

**解决方案**：
- 使用 `Time<Virtual>::set_relative_speed()` 设置相对速度
- 使用 `Time<Virtual>::pause()` 暂停时间
- 使用 `Time<Virtual>::unpause()` 恢复时间

### 性能考虑

1. **时间资源**：时间资源是轻量级的，可以频繁访问
2. **定时器**：定时器更新是高效的，可以大量使用
3. **虚拟时间**：虚拟时间计算是高效的，可以用于控制游戏速度

## 相关资源

**相关源代码文件**：
- `bevy/examples/time/time.rs` - 时间系统基础示例
- `bevy/examples/time/timers.rs` - 定时器示例
- `bevy/examples/time/virtual_time.rs` - 虚拟时间示例

**官方文档链接**：
- [Bevy 时间系统](https://docs.rs/bevy/latest/bevy/time/index.html)
- [Bevy 时间示例](https://github.com/bevyengine/bevy/tree/main/examples/time)

**进一步学习建议**：
- 学习系统调度，了解固定时间步的使用
- 学习资源管理，了解资源的生命周期

---

**索引**：[返回上级目录](/wiki/BevyBook/ECS/)
