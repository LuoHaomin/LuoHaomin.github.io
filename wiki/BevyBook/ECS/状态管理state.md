# 状态管理（State）## 概述

**学习目标**：
- 理解 Bevy 状态管理的基本概念
- 掌握基本状态的使用
- 了解子状态和计算状态
- 学会使用自定义状态转换

**前置知识要求**：
- Bevy 快速入门
- ECS 基础
- 系统（Systems）
- 系统调度（Schedule & App）


## 核心概念

### 什么是状态管理？

状态管理是 Bevy 中用于控制应用流程的功能。状态管理允许您定义应用的不同状态（如菜单、游戏中、暂停等），并根据状态控制系统的执行。

**为什么需要状态管理？**

1. **应用控制流**：状态管理可以控制应用的不同阶段
2. **系统组织**：状态管理可以组织和管理系统
3. **资源管理**：状态管理可以管理不同状态的资源
4. **代码清晰**：状态管理可以使代码更清晰、更易维护

### 状态管理的核心组件

Bevy 状态管理包含以下核心组件：

- **States**：状态 trait，用于定义状态类型
- **State<T>**：状态资源，用于读取当前状态
- **NextState<T>**：下一状态资源，用于设置下一状态
- **OnEnter<T>**：进入状态时的调度
- **OnExit<T>**：退出状态时的调度

## 基础用法

### 基本状态

创建和使用基本状态。

**源代码文件**：`bevy/examples/state/states.rs`

**代码示例**：

```rust
use bevy::{dev_tools::states::*, prelude::*};

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .init_state::<AppState>() // 或者我们可以使用 .insert_state(AppState::Menu)
        .add_systems(Startup, setup)
        // 此系统在我们进入 `AppState::Menu` 时运行，在 `StateTransition` 调度期间。
        // 首先运行我们离开的状态的退出调度中的所有系统，
        // 然后运行我们进入的状态的进入调度中的所有系统。
        .add_systems(OnEnter(AppState::Menu), setup_menu)
        // 相比之下，更新系统存储在 `Update` 调度中。它们只是
        // 检查 `State<T>` 资源的值，以查看是否应该在每帧运行。
        .add_systems(Update, menu.run_if(in_state(AppState::Menu)))
        .add_systems(OnExit(AppState::Menu), cleanup_menu)
        .add_systems(OnEnter(AppState::InGame), setup_game)
        .add_systems(
            Update,
            (movement, change_color).run_if(in_state(AppState::InGame)),
        )
        .add_systems(Update, log_transitions::<AppState>)
        .run();
}

#[derive(Debug, Clone, Copy, Default, Eq, PartialEq, Hash, States)]
enum AppState {
    #[default]
    Menu,
    InGame,
}

#[derive(Resource)]
struct MenuData {
    button_entity: Entity,
}

fn setup(mut commands: Commands) {
    commands.spawn(Camera2d);
}

fn setup_menu(mut commands: Commands) {
    let button_entity = commands
        .spawn((
            Node {
                // 居中按钮
                width: percent(100),
                height: percent(100),
                justify_content: JustifyContent::Center,
                align_items: AlignItems::Center,
                ..default()
            },
            children![(
                Button,
                Node {
                    width: px(150),
                    height: px(65),
                    // 水平居中子文本
                    justify_content: JustifyContent::Center,
                    // 垂直居中子文本
                    align_items: AlignItems::Center,
                    ..default()
                },
                BackgroundColor(NORMAL_BUTTON),
                children![(
                    Text::new("Play"),
                    TextFont {
                        font_size: 33.0,
                        ..default()
                    },
                    TextColor(Color::srgb(0.9, 0.9, 0.9)),
                )],
            )],
        ))
        .id();
    commands.insert_resource(MenuData { button_entity });
}

fn menu(
    mut next_state: ResMut<NextState<AppState>>,
    mut interaction_query: Query<
        (&Interaction, &mut BackgroundColor),
        (Changed<Interaction>, With<Button>),
    >,
) {
    for (interaction, mut color) in &mut interaction_query {
        match *interaction {
            Interaction::Pressed => {
                *color = PRESSED_BUTTON.into();
                next_state.set(AppState::InGame);
            }
            Interaction::Hovered => {
                *color = HOVERED_BUTTON.into();
            }
            Interaction::None => {
                *color = NORMAL_BUTTON.into();
            }
        }
    }
}

fn cleanup_menu(mut commands: Commands, menu_data: Res<MenuData>) {
    commands.entity(menu_data.button_entity).despawn();
}

fn setup_game(mut commands: Commands, asset_server: Res<AssetServer>) {
    commands.spawn(Sprite::from_image(asset_server.load("branding/icon.png")));
}
```

**关键要点**：
- 使用 `#[derive(States)]` 创建状态类型
- 使用 `init_state::<T>()` 初始化状态
- 使用 `OnEnter<T>` 在进入状态时运行系统
- 使用 `OnExit<T>` 在退出状态时运行系统
- 使用 `in_state(T)` 条件运行系统
- 使用 `NextState<T>` 设置下一状态

**说明**：
基本状态是状态管理的基础。通过使用基本状态，可以控制应用的不同阶段，并根据状态控制系统的执行。

## 进阶用法

### 子状态（SubStates）

使用子状态创建更复杂的状态模式。

**源代码文件**：`bevy/examples/state/sub_states.rs`

**代码示例**：

```rust
use bevy::{dev_tools::states::*, prelude::*};

#[derive(Debug, Clone, Copy, Default, Eq, PartialEq, Hash, States)]
enum AppState {
    #[default]
    Menu,
    InGame,
}

// 在这种情况下，我们不是派生 `States`，而是派生 `SubStates`
#[derive(Debug, Clone, Copy, Default, Eq, PartialEq, Hash, SubStates)]
// 我们需要添加一个属性来告诉我们源状态是什么
// 以及它需要什么值。这将确保除非我们
// 在 [`AppState::InGame`] 中，否则 [`IsPaused`] 状态资源
// 将不存在。
#[source(AppState = AppState::InGame)]
#[states(scoped_entities)]
enum IsPaused {
    #[default]
    Running,
    Paused,
}

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .init_state::<AppState>()
        .add_sub_state::<IsPaused>() // 我们在这里设置子状态
        .add_systems(Startup, setup)
        .add_systems(OnEnter(AppState::Menu), setup_menu)
        .add_systems(Update, menu.run_if(in_state(AppState::Menu)))
        .add_systems(OnExit(AppState::Menu), cleanup_menu)
        .add_systems(OnEnter(AppState::InGame), setup_game)
        .add_systems(OnEnter(IsPaused::Paused), setup_paused_screen)
        .add_systems(
            Update,
            (
                // 这里我们不是依赖 [`AppState::InGame`]，而是依赖
                // [`IsPaused::Running`]，因为我们不希望在暂停时移动或改变颜色
                (movement, change_color).run_if(in_state(IsPaused::Running)),
                // 另一方面，暂停切换需要无论是否暂停都能工作，
                // 所以它使用 [`AppState::InGame`] 而不是。
                toggle_pause.run_if(in_state(AppState::InGame)),
            ),
        )
        .add_systems(Update, log_transitions::<AppState>)
        .run();
}
```

**关键要点**：
- 使用 `#[derive(SubStates)]` 创建子状态
- 使用 `#[source(AppState = AppState::InGame)]` 指定源状态
- 使用 `add_sub_state::<T>()` 添加子状态
- 子状态只在源状态存在时存在

**说明**：
子状态是状态管理的高级功能。通过使用子状态，可以创建更复杂的状态模式，同时依赖简单的枚举。

### 计算状态（ComputedStates）

使用计算状态创建更复杂的状态模式。

**源代码文件**：`bevy/examples/state/computed_states.rs`

**关键信息**：
- 使用 `ComputedStates` trait 创建计算状态
- 使用 `SourceStates` 指定源状态
- 使用 `compute` 函数计算状态值
- 计算状态可以从多个源状态计算

**说明**：
计算状态是状态管理的高级功能。通过使用计算状态，可以创建从其他状态计算得出的状态，实现更灵活的状态管理。

### 自定义状态转换

创建自定义状态转换行为。

**源代码文件**：`bevy/examples/state/custom_transitions.rs`

**关键信息**：
- 使用 `StateTransition` 调度处理状态转换
- 使用 `StateTransitionEvent` 监听状态转换事件
- 使用 `OnReenter` 和 `OnReexit` 创建自定义转换
- 使用 `IdentityTransitionsPlugin` 注册自定义转换

**说明**：
自定义状态转换是状态管理的高级功能。通过使用自定义状态转换，可以实现更复杂的状态转换行为。

## 实际应用

### 在游戏开发中的应用场景

状态管理在游戏开发中有广泛的应用：

1. **游戏流程**：控制游戏的不同阶段（菜单、游戏中、暂停等）
2. **系统组织**：根据状态组织和管理系统
3. **资源管理**：管理不同状态的资源
4. **代码清晰**：使代码更清晰、更易维护

### 常见问题

**问题 1**：如何创建状态？

**解决方案**：
- 使用 `#[derive(States)]` 创建状态类型
- 使用 `init_state::<T>()` 初始化状态
- 使用 `insert_state(T)` 插入初始状态

**问题 2**：如何切换状态？

**解决方案**：
- 使用 `NextState<T>::set()` 设置下一状态
- 使用 `NextState<T>::set_if_neq()` 仅在状态不同时设置
- 状态转换在 `StateTransition` 调度中处理

**问题 3**：如何根据状态运行系统？

**解决方案**：
- 使用 `in_state(T)` 条件运行系统
- 使用 `OnEnter<T>` 在进入状态时运行系统
- 使用 `OnExit<T>` 在退出状态时运行系统

### 性能考虑

1. **状态转换**：状态转换在 `StateTransition` 调度中处理
2. **系统条件**：使用 `in_state` 条件可以减少不必要的系统执行
3. **子状态**：子状态只在源状态存在时存在，可以减少资源使用

## 相关资源

**相关源代码文件**：
- `bevy/examples/state/states.rs` - 基本状态示例
- `bevy/examples/state/sub_states.rs` - 子状态示例
- `bevy/examples/state/computed_states.rs` - 计算状态示例
- `bevy/examples/state/custom_transitions.rs` - 自定义状态转换示例

**官方文档链接**：
- [Bevy 状态管理](https://docs.rs/bevy/latest/bevy/app/trait.States.html)
- [Bevy 状态示例](https://github.com/bevyengine/bevy/tree/main/examples/state)

**进一步学习建议**：
- 学习系统调度，了解系统执行顺序
- 学习 ECS 进阶，了解高级功能

---

**索引**：[返回上级目录](/wiki/BevyBook/ECS/)



