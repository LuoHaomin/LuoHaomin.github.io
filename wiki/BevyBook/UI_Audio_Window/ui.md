# 用户界面（UI）## 概述

**学习目标**：
- 理解 UI 系统的基本概念
- 掌握按钮的创建和使用
- 学会创建和更新文本
- 了解 Flexbox 和 Grid 布局
- 掌握 UI 样式和交互

**前置知识要求**：
- Bevy 快速入门
- ECS 基础
- 窗口管理基础


## 核心概念

### 什么是 UI 系统？

UI 系统是 Bevy 中用于创建用户界面的功能。Bevy 的 UI 系统支持按钮、文本、布局、样式等多种 UI 元素。

**为什么需要 UI 系统？**

1. **用户交互**：UI 系统提供用户交互界面
2. **信息显示**：UI 系统可以显示游戏信息
3. **菜单系统**：UI 系统可以创建菜单和设置界面
4. **HUD**：UI 系统可以创建游戏 HUD

### UI 系统的核心组件

Bevy UI 系统包含以下核心组件：

- **Node**：UI 节点，用于布局和样式
- **Button**：按钮组件，用于用户交互
- **Text**：文本组件，用于显示文本
- **BackgroundColor**：背景颜色组件
- **BorderColor**：边框颜色组件
- **Interaction**：交互状态组件

## 基础用法

### 创建按钮

创建和使用按钮。

**源代码文件**：`bevy/examples/ui/button.rs`

**代码示例**：

```rust
use bevy::prelude::*;

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .init_resource::<InputFocus>()
        .add_systems(Startup, setup)
        .add_systems(Update, button_system)
        .run();
}

const NORMAL_BUTTON: Color = Color::srgb(0.15, 0.15, 0.15);
const HOVERED_BUTTON: Color = Color::srgb(0.25, 0.25, 0.25);
const PRESSED_BUTTON: Color = Color::srgb(0.35, 0.75, 0.35);

fn button_system(
    mut input_focus: ResMut<InputFocus>,
    mut interaction_query: Query<
        (
            Entity,
            &Interaction,
            &mut BackgroundColor,
            &mut BorderColor,
            &mut Button,
            &Children,
        ),
        Changed<Interaction>,
    >,
    mut text_query: Query<&mut Text>,
) {
    for (entity, interaction, mut color, mut border_color, mut button, children) in
        &mut interaction_query
    {
        let mut text = text_query.get_mut(children[0]).unwrap();

        match *interaction {
            Interaction::Pressed => {
                input_focus.set(entity);
                **text = "Press".to_string();
                *color = PRESSED_BUTTON.into();
                *border_color = BorderColor::all(RED);
                button.set_changed();
            }
            Interaction::Hovered => {
                input_focus.set(entity);
                **text = "Hover".to_string();
                *color = HOVERED_BUTTON.into();
                *border_color = BorderColor::all(Color::WHITE);
                button.set_changed();
            }
            Interaction::None => {
                input_focus.clear();
                **text = "Button".to_string();
                *color = NORMAL_BUTTON.into();
                *border_color = BorderColor::all(Color::BLACK);
            }
        }
    }
}

fn setup(mut commands: Commands, assets: Res<AssetServer>) {
    commands.spawn(Camera2d);
    commands.spawn(button(&assets));
}

fn button(asset_server: &AssetServer) -> impl Bundle {
    (
        Node {
            width: percent(100),
            height: percent(100),
            align_items: AlignItems::Center,
            justify_content: JustifyContent::Center,
            ..default()
        },
        children![(
            Button,
            Node {
                width: px(150),
                height: px(65),
                border: UiRect::all(px(5)),
                justify_content: JustifyContent::Center,
                align_items: AlignItems::Center,
                ..default()
            },
            BorderColor::all(Color::WHITE),
            BorderRadius::MAX,
            BackgroundColor(Color::BLACK),
            children![(
                Text::new("Button"),
                TextFont {
                    font: asset_server.load("fonts/FiraSans-Bold.ttf"),
                    font_size: 40.0,
                    ..default()
                },
            )],
        )],
    )
}
```

**关键要点**：
- 使用 `Button` 组件创建按钮
- 使用 `Interaction` 组件检测交互状态
- 可以响应 `Pressed`、`Hovered`、`None` 三种交互状态
- 需要设置 `InputFocus` 资源以支持无障碍功能

**说明**：
按钮是 UI 系统中最常用的交互元素。通过检测交互状态，可以创建响应式的按钮效果。

### 创建文本

创建和更新文本。

**源代码文件**：`bevy/examples/ui/text.rs`

**代码示例**：

```rust
use bevy::prelude::*;

fn setup(mut commands: Commands, asset_server: Res<AssetServer>) {
    commands.spawn(Camera2d);
    
    // 创建单个文本
    commands.spawn((
        Text::new("hello\nbevy!"),
        TextFont {
            font: asset_server.load("fonts/FiraSans-Bold.ttf"),
            font_size: 67.0,
            ..default()
        },
        TextShadow::default(),
        TextLayout::new_with_justify(Justify::Center),
        Node {
            position_type: PositionType::Absolute,
            bottom: px(5),
            right: px(5),
            ..default()
        },
    ));
    
    // 创建多段文本
    commands
        .spawn((
            Text::new("FPS: "),
            TextFont {
                font: asset_server.load("fonts/FiraSans-Bold.ttf"),
                font_size: 42.0,
                ..default()
            },
        ))
        .with_child((
            TextSpan::default(),
            TextFont {
                font_size: 33.0,
                ..default()
            },
            TextColor(GOLD.into()),
        ));
}

fn text_update_system(
    diagnostics: Res<DiagnosticsStore>,
    mut query: Query<&mut Text, With<FpsText>>,
) {
    for mut text in &mut query {
        if let Some(fps) = diagnostics.get(&FrameTimeDiagnosticsPlugin::FPS) {
            if let Some(value) = fps.smoothed() {
                text.0 = format!("{value:.2}");
            }
        }
    }
}
```

**关键要点**：
- 使用 `Text` 组件创建文本
- 可以使用 `TextFont` 设置字体和大小
- 可以使用 `TextColor` 设置文本颜色
- 可以使用 `TextSpan` 创建多段文本
- 可以在系统中更新文本内容

**说明**：
文本是 UI 系统中用于显示信息的重要元素。通过更新文本内容，可以显示游戏状态、分数等信息。

### Flexbox 布局

使用 Flexbox 布局 UI 元素。

**源代码文件**：`bevy/examples/ui/flex_layout.rs`

**代码示例**：

```rust
use bevy::prelude::*;

fn spawn_layout(mut commands: Commands, asset_server: Res<AssetServer>) {
    let font = asset_server.load("fonts/FiraSans-Bold.ttf");
    commands.spawn(Camera2d);
    
    commands
        .spawn((
            Node {
                width: percent(100),
                height: percent(100),
                flex_direction: FlexDirection::Column,
                align_items: AlignItems::Center,
                padding: UiRect::all(Val::Px(12.0)),
                row_gap: Val::Px(12.0),
                ..Default::default()
            },
            BackgroundColor(Color::BLACK),
        ))
        .with_children(|builder| {
            builder
                .spawn(Node {
                    flex_direction: FlexDirection::Row,
                    ..default()
                })
                .with_children(|builder| {
                    // 添加子元素
                });
        });
}
```

**关键要点**：
- 使用 `Node` 组件创建布局容器
- 使用 `flex_direction` 设置布局方向（`Column` 或 `Row`）
- 使用 `align_items` 设置对齐方式
- 使用 `justify_content` 设置内容分布
- 使用 `padding`、`row_gap`、`column_gap` 设置间距

**说明**：
Flexbox 布局是创建响应式 UI 的重要工具。通过设置不同的布局属性，可以创建各种 UI 布局。

## 进阶用法

### Grid 布局

使用 Grid 布局 UI 元素。

**源代码文件**：`bevy/examples/ui/grid.rs`

**关键信息**：
- 可以使用 `Grid` 组件创建网格布局
- 可以设置网格的行和列
- 可以设置网格项的跨行和跨列
- 可以设置网格间距和对齐方式

**说明**：
Grid 布局适合创建表格、仪表板等复杂的 UI 布局。

### UI 样式

设置 UI 元素的样式。

**关键信息**：
- 可以使用 `BackgroundColor` 设置背景颜色
- 可以使用 `BorderColor` 设置边框颜色
- 可以使用 `BorderRadius` 设置圆角
- 可以使用 `TextShadow` 设置文本阴影
- 可以使用 `TextLayout` 设置文本布局

**说明**：
UI 样式可以让界面更加美观和易用。通过设置不同的样式属性，可以创建各种视觉效果。

### UI 交互

处理 UI 元素的交互。

**关键信息**：
- 使用 `Interaction` 组件检测交互状态
- 可以响应 `Pressed`、`Hovered`、`None` 三种状态
- 可以使用 `InputFocus` 资源支持无障碍功能
- 可以使用 `Button` 组件的 `set_changed()` 方法更新状态

**说明**：
UI 交互是创建响应式界面的关键。通过检测交互状态，可以创建各种交互效果。

## 实际应用

### 在游戏开发中的应用场景

UI 系统在游戏开发中有广泛的应用：

1. **游戏菜单**：创建主菜单、设置菜单等
2. **HUD**：创建游戏 HUD，显示分数、生命值等
3. **对话框**：创建对话和提示界面
4. **设置界面**：创建设置和配置界面

### 常见问题

**问题 1**：如何创建响应式布局？

**解决方案**：使用 Flexbox 或 Grid 布局，并使用百分比或相对单位设置大小。

**问题 2**：如何处理 UI 点击事件？

**解决方案**：使用 `Interaction` 组件检测交互状态，并在系统中处理交互逻辑。

**问题 3**：如何更新 UI 文本？

**解决方案**：在系统中查询文本组件，并使用 `Text` 的 `set()` 方法更新内容。

### 性能考虑

1. **UI 元素数量**：尽量减少 UI 元素数量以提高性能
2. **文本更新频率**：避免频繁更新文本内容
3. **布局计算**：合理使用布局属性以减少计算开销
4. **交互检测**：只在需要时检测交互状态

## 相关资源

**相关源代码文件**：
- `bevy/examples/ui/button.rs` - 按钮示例
- `bevy/examples/ui/text.rs` - 文本示例
- `bevy/examples/ui/flex_layout.rs` - Flexbox 布局示例
- `bevy/examples/ui/grid.rs` - Grid 布局示例
- `bevy/examples/ui/directional_navigation.rs` - 方向导航示例

**官方文档链接**：
- [Bevy UI 官方文档](https://docs.rs/bevy_ui/latest/bevy_ui/)
- [UI 示例](https://github.com/bevyengine/bevy/tree/main/examples/ui)

**进一步学习建议**：
- 学习窗口管理，了解如何在窗口中创建 UI
- 学习输入处理，了解如何处理 UI 输入事件
- 学习动画系统，了解如何对 UI 元素进行动画处理

---

**索引**：[返回上级目录](/wiki/bevybook/ui_audio_window/)


