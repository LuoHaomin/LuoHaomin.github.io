# input基础# Bevy Input 基础教程

本教程基于Bevy官方示例，按主题组织，提供易于理解的输入系统使用参考。

## 键盘输入

### 基础键盘输入

**示例文件**: `keyboard_input.rs`

最基本的键盘输入处理：

```rust
use bevy::prelude::*;

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Update, keyboard_input_system)
        .run();
}

/// 处理键盘输入的系统
fn keyboard_input_system(keyboard_input: Res<ButtonInput<KeyCode>>) {
    // 检查按键是否正在被按下
    if keyboard_input.pressed(KeyCode::KeyA) {
        info!("'A' 键正在被按下");
    }

    // 检查按键是否刚刚被按下
    if keyboard_input.just_pressed(KeyCode::KeyA) {
        info!("'A' 键刚刚被按下");
    }

    // 检查按键是否刚刚被释放
    if keyboard_input.just_released(KeyCode::KeyA) {
        info!("'A' 键刚刚被释放");
    }
}

```

**关键要点**:

- 使用 `Res<ButtonInput<KeyCode>>` 获取键盘输入状态
- `pressed()` 检查按键是否正在被按下
- `just_pressed()` 检查按键是否刚刚被按下（只在按下瞬间触发一次）
- `just_released()` 检查按键是否刚刚被释放

### 键盘事件监听

**示例文件**: `keyboard_input_events.rs`

监听所有键盘事件：

```rust
use bevy::{input::keyboard::KeyboardInput, prelude::*};

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Update, print_keyboard_event_system)
        .run();
}

/// 打印所有键盘事件的系统
fn print_keyboard_event_system(mut keyboard_input_events: EventReader<KeyboardInput>) {
    for event in keyboard_input_events.read() {
        info!("键盘事件: {:?}", event);
    }
}

```

**关键要点**:

- 使用 `EventReader<KeyboardInput>` 监听键盘事件
- 事件包含按键状态、物理键、逻辑键等信息
- 适合需要详细键盘信息的场景

### 键盘修饰键

**示例文件**: `keyboard_modifiers.rs`

处理组合键和修饰键：

```rust
use bevy::prelude::*;

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Update, keyboard_input_system)
        .run();
}

/// 处理组合键的系统
fn keyboard_input_system(input: Res<ButtonInput<KeyCode>>) {
    // 检查Shift键是否被按下（左Shift或右Shift）
    let shift = input.any_pressed([KeyCode::ShiftLeft, KeyCode::ShiftRight]);

    // 检查Ctrl键是否被按下（左Ctrl或右Ctrl）
    let ctrl = input.any_pressed([KeyCode::ControlLeft, KeyCode::ControlRight]);

    // 检查组合键 Ctrl + Shift + A
    if ctrl && shift && input.just_pressed(KeyCode::KeyA) {
        info!("刚刚按下了 Ctrl + Shift + A!");
    }

    // 检查其他组合键
    if ctrl && input.just_pressed(KeyCode::KeyS) {
        info!("保存快捷键被触发");
    }

    if ctrl && input.just_pressed(KeyCode::KeyZ) {
        info!("撤销快捷键被触发");
    }
}

```

**关键要点**:

- `any_pressed()` 检查多个按键中是否有任意一个被按下
- 支持左右修饰键的检测
- 适合实现快捷键和组合键功能

### 字符输入

**示例文件**: `char_input_events.rs`

处理字符输入（支持多语言）：

```rust
use bevy::{
    input::keyboard::{Key, KeyboardInput},
    prelude::*,
};

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Update, print_char_event_system)
        .run();
}

/// 处理字符输入的系统
fn print_char_event_system(mut char_input_events: EventReader<KeyboardInput>) {
    for event in char_input_events.read() {
        // 只处理按键按下事件
        if !event.state.is_pressed() {
            continue;
        }

        // 检查是否为字符输入
        if let Key::Character(character) = &event.logical_key {
            info!("输入字符: '{}'", character);
        }
    }
}

```

**关键要点**:

- 使用 `event.logical_key` 获取逻辑键值
- `Key::Character` 表示字符输入
- 支持多语言字符输入
- 适合文本输入和聊天功能

---

## 鼠标输入

### 基础鼠标输入

**示例文件**: `mouse_input.rs`

处理鼠标按钮和移动：

```rust
use bevy::{
    input::mouse::{AccumulatedMouseMotion, AccumulatedMouseScroll},
    prelude::*,
};

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Update, (mouse_click_system, mouse_move_system))
        .run();
}

/// 处理鼠标点击的系统
fn mouse_click_system(mouse_button_input: Res<ButtonInput<MouseButton>>) {
    // 检查左键是否正在被按下
    if mouse_button_input.pressed(MouseButton::Left) {
        info!("左键正在被按下");
    }

    // 检查左键是否刚刚被按下
    if mouse_button_input.just_pressed(MouseButton::Left) {
        info!("左键刚刚被按下");
    }

    // 检查左键是否刚刚被释放
    if mouse_button_input.just_released(MouseButton::Left) {
        info!("左键刚刚被释放");
    }

    // 检查右键
    if mouse_button_input.just_pressed(MouseButton::Right) {
        info!("右键刚刚被按下");
    }

    // 检查中键
    if mouse_button_input.just_pressed(MouseButton::Middle) {
        info!("中键刚刚被按下");
    }
}

/// 处理鼠标移动和滚轮的系统
fn mouse_move_system(
    accumulated_mouse_motion: Res<AccumulatedMouseMotion>,
    accumulated_mouse_scroll: Res<AccumulatedMouseScroll>,
) {
    // 处理鼠标移动
    if accumulated_mouse_motion.delta != Vec2::ZERO {
        let delta = accumulated_mouse_motion.delta;
        info!("鼠标移动了 ({}, {})", delta.x, delta.y);
    }

    // 处理鼠标滚轮
    if accumulated_mouse_scroll.delta != Vec2::ZERO {
        let delta = accumulated_mouse_scroll.delta;
        info!("鼠标滚动了 ({}, {})", delta.x, delta.y);
    }
}

```

**关键要点**:

- `ButtonInput<MouseButton>` 处理鼠标按钮
- `AccumulatedMouseMotion` 处理鼠标移动
- `AccumulatedMouseScroll` 处理滚轮滚动
- 支持左键、右键、中键检测

### 鼠标事件监听

**示例文件**: `mouse_input_events.rs`

监听详细的鼠标事件：

```rust
use bevy::{
    input::{
        gestures::*,
        mouse::{MouseButtonInput, MouseMotion, MouseWheel},
    },
    prelude::*,
};

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Update, print_mouse_events_system)
        .run();
}

/// 打印所有鼠标事件的系统
fn print_mouse_events_system(
    mut mouse_button_input_events: EventReader<MouseButtonInput>,
    mut mouse_motion_events: EventReader<MouseMotion>,
    mut cursor_moved_events: EventReader<CursorMoved>,
    mut mouse_wheel_events: EventReader<MouseWheel>,
    mut pinch_gesture_events: EventReader<PinchGesture>,
    mut rotation_gesture_events: EventReader<RotationGesture>,
    mut double_tap_gesture_events: EventReader<DoubleTapGesture>,
) {
    // 鼠标按钮事件
    for event in mouse_button_input_events.read() {
        info!("鼠标按钮事件: {:?}", event);
    }

    // 鼠标移动事件
    for event in mouse_motion_events.read() {
        info!("鼠标移动事件: {:?}", event);
    }

    // 光标移动事件
    for event in cursor_moved_events.read() {
        info!("光标移动事件: {:?}", event);
    }

    // 鼠标滚轮事件
    for event in mouse_wheel_events.read() {
        info!("鼠标滚轮事件: {:?}", event);
    }

    // 手势事件（仅macOS）
    for event in pinch_gesture_events.read() {
        info!("捏合手势事件: {:?}", event);
    }

    for event in rotation_gesture_events.read() {
        info!("旋转手势事件: {:?}", event);
    }

    for event in double_tap_gesture_events.read() {
        info!("双击手势事件: {:?}", event);
    }
}

```

**关键要点**:

- 多种事件类型提供不同粒度的鼠标信息
- 手势事件仅在macOS上可用
- 适合需要精确鼠标控制的场景

### 鼠标光标控制

**示例文件**: `mouse_grab.rs`

控制鼠标光标的显示和锁定：

```rust
use bevy::{prelude::*, window::CursorGrabMode};

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Update, grab_mouse)
        .run();
}

/// 控制鼠标光标抓取的系统
fn grab_mouse(
    mut window: Single<&mut Window>,
    mouse: Res<ButtonInput<MouseButton>>,
    key: Res<ButtonInput<KeyCode>>,
) {
    // 左键点击时隐藏光标并锁定鼠标
    if mouse.just_pressed(MouseButton::Left) {
        window.cursor_options.visible = false;
        window.cursor_options.grab_mode = CursorGrabMode::Locked;
        info!("鼠标已锁定");
    }

    // ESC键释放鼠标
    if key.just_pressed(KeyCode::Escape) {
        window.cursor_options.visible = true;
        window.cursor_options.grab_mode = CursorGrabMode::None;
        info!("鼠标已释放");
    }

    // 其他光标模式示例
    if key.just_pressed(KeyCode::Key1) {
        // 限制光标在窗口内
        window.cursor_options.grab_mode = CursorGrabMode::Confined;
    }

    if key.just_pressed(KeyCode::Key2) {
        // 隐藏光标但不锁定
        window.cursor_options.visible = false;
        window.cursor_options.grab_mode = CursorGrabMode::None;
    }
}

```

**关键要点**:

- `CursorGrabMode::Locked` 锁定鼠标到窗口中心
- `CursorGrabMode::Confined` 限制鼠标在窗口内
- `CursorGrabMode::None` 正常模式
- 适合FPS游戏等需要鼠标控制的场景

---

## 触摸输入

### 基础触摸输入

**示例文件**: `touch_input.rs`

处理触摸屏输入：

```rust
use bevy::{input::touch::*, prelude::*};

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Update, touch_system)
        .run();
}

fn touch_system(touches: Res<Touches>) {
    // 处理刚刚按下的触摸
    for touch in touches.iter_just_pressed() {
        info!(
            "触摸按下 - ID: {}, 位置: {}",
            touch.id(),
            touch.position()
        );
    }

    // 处理刚刚释放的触摸
    for touch in touches.iter_just_released() {
        info!(
            "触摸释放 - ID: {}, 位置: {}",
            touch.id(),
            touch.position()
        );
    }

    // 处理被取消的触摸
    for touch in touches.iter_just_canceled() {
        info!("触摸取消 - ID: {}", touch.id());
    }

    // 处理所有当前活动的触摸
    for touch in touches.iter() {
        info!("活动触摸: {touch:?}");
        info!("  是否刚刚按下: {}", touches.just_pressed(touch.id()));
        info!("  是否正在按下: {}", touches.pressed(touch.id()));
        info!("  位置: {}", touch.position());
        info!("  压力: {}", touch.force().unwrap_or(0.0));
    }

    // 获取触摸数量
    info!("当前触摸数量: {}", touches.count());
}

```

**关键要点**:

- `Touches` 资源提供触摸状态
- 每个触摸有唯一ID
- 支持多点触摸
- 包含位置和压力信息

### 触摸事件监听

**示例文件**: `touch_input_events.rs`

监听详细的触摸事件：

```rust
use bevy::{input::touch::*, prelude::*};

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Update, touch_event_system)
        .run();
}

fn touch_event_system(mut touch_events: EventReader<TouchInput>) {
    for event in touch_events.read() {
        info!("触摸事件: {:?}", event);

        match event.phase {
            TouchPhase::Started => {
                info!("触摸开始 - ID: {}, 位置: {}", event.id, event.position);
            }
            TouchPhase::Moved => {
                info!("触摸移动 - ID: {}, 位置: {}", event.id, event.position);
            }
            TouchPhase::Ended => {
                info!("触摸结束 - ID: {}, 位置: {}", event.id, event.position);
            }
            TouchPhase::Canceled => {
                info!("触摸取消 - ID: {}", event.id);
            }
        }
    }
}

```

**关键要点**:

- `TouchInput` 事件提供详细的触摸信息
- `TouchPhase` 表示触摸的不同阶段
- 适合需要精确触摸控制的场景

---

## 游戏手柄输入

### 基础游戏手柄输入

**示例文件**: `gamepad_input.rs`

处理游戏手柄输入：

```rust
use bevy::prelude::*;

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Update, gamepad_system)
        .run();
}

fn gamepad_system(gamepads: Query<(Entity, &Gamepad)>) {
    for (entity, gamepad) in &gamepads {
        // 处理按钮输入
        if gamepad.just_pressed(GamepadButton::South) {
            info!("手柄 {} 刚刚按下 South 按钮", entity);
        } else if gamepad.just_released(GamepadButton::South) {
            info!("手柄 {} 刚刚释放 South 按钮", entity);
        }

        // 处理扳机键（模拟输入）
        let right_trigger = gamepad.get(GamepadButton::RightTrigger2).unwrap();
        if right_trigger.abs() > 0.01 {
            info!("手柄 {} 右扳机值: {}", entity, right_trigger);
        }

        // 处理摇杆输入
        let left_stick_x = gamepad.get(GamepadAxis::LeftStickX).unwrap();
        let left_stick_y = gamepad.get(GamepadAxis::LeftStickY).unwrap();
        if left_stick_x.abs() > 0.01 || left_stick_y.abs() > 0.01 {
            info!("手柄 {} 左摇杆: ({}, {})", entity, left_stick_x, left_stick_y);
        }

        let right_stick_x = gamepad.get(GamepadAxis::RightStickX).unwrap();
        let right_stick_y = gamepad.get(GamepadAxis::RightStickY).unwrap();
        if right_stick_x.abs() > 0.01 || right_stick_y.abs() > 0.01 {
            info!("手柄 {} 右摇杆: ({}, {})", entity, right_stick_x, right_stick_y);
        }

        // 处理其他按钮
        if gamepad.just_pressed(GamepadButton::North) {
            info!("手柄 {} 按下 North 按钮", entity);
        }
        if gamepad.just_pressed(GamepadButton::East) {
            info!("手柄 {} 按下 East 按钮", entity);
        }
        if gamepad.just_pressed(GamepadButton::West) {
            info!("手柄 {} 按下 West 按钮", entity);
        }
    }
}

```

**关键要点**:

- `Gamepad` 组件提供手柄输入状态
- 支持多个手柄同时连接
- 按钮有数字和模拟两种输入
- 摇杆提供X、Y轴模拟输入

### 游戏手柄事件监听

**示例文件**: `gamepad_input_events.rs`

监听游戏手柄连接和输入事件：

```rust
use bevy::{
    input::gamepad::{
        GamepadAxisChangedEvent, GamepadButtonChangedEvent, GamepadButtonStateChangedEvent,
        GamepadConnectionEvent, GamepadEvent,
    },
    prelude::*,
};

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Update, (gamepad_events, gamepad_ordered_events))
        .run();
}

fn gamepad_events(
    mut connection_events: EventReader<GamepadConnectionEvent>,
    mut axis_changed_events: EventReader<GamepadAxisChangedEvent>,
    mut button_changed_events: EventReader<GamepadButtonChangedEvent>,
    mut button_input_events: EventReader<GamepadButtonStateChangedEvent>,
) {
    // 处理连接事件
    for connection_event in connection_events.read() {
        info!("手柄连接事件: {:?}", connection_event);
    }

    // 处理轴变化事件
    for axis_changed_event in axis_changed_events.read() {
        info!(
            "手柄轴变化 - 轴: {:?}, 手柄: {}, 值: {}",
            axis_changed_event.axis, axis_changed_event.entity, axis_changed_event.value
        );
    }

    // 处理按钮变化事件
    for button_changed_event in button_changed_events.read() {
        info!(
            "手柄按钮变化 - 按钮: {:?}, 手柄: {}, 值: {}",
            button_changed_event.button, button_changed_event.entity, button_changed_event.value
        );
    }

    // 处理按钮状态变化事件
    for button_input_event in button_input_events.read() {
        info!("手柄按钮状态变化: {:?}", button_input_event);
    }
}

// 处理有序的游戏手柄事件
fn gamepad_ordered_events(mut gamepad_events: EventReader<GamepadEvent>) {
    for gamepad_event in gamepad_events.read() {
        match gamepad_event {
            GamepadEvent::Connection(connection_event) => {
                info!("手柄连接事件: {:?}", connection_event);
            }
            GamepadEvent::Button(button_event) => {
                info!("手柄按钮事件: {:?}", button_event);
            }
            GamepadEvent::Axis(axis_event) => {
                info!("手柄轴事件: {:?}", axis_event);
            }
        }
    }
}

```

**关键要点**:

- 多种事件类型提供不同粒度的手柄信息
- 连接事件处理手柄插拔
- 轴和按钮事件提供精确的输入变化
- 有序事件确保事件处理的正确顺序

---

## 文本输入

### 基础文本输入

**示例文件**: `text_input.rs`

处理文本输入和IME（输入法编辑器）：

```rust
use std::mem;
use bevy::{
    input::keyboard::{Key, KeyboardInput},
    prelude::*,
};

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Startup, setup_scene)
        .add_systems(
            Update,
            (
                toggle_ime,
                listen_ime_events,
                listen_keyboard_input_events,
                bubbling_text,
            ),
        )
        .run();
}

fn setup_scene(mut commands: Commands, asset_server: Res<AssetServer>) {
    commands.spawn(Camera2d);

    // 使用支持更多字符的字体
    let font = asset_server.load("fonts/FiraMono-Medium.ttf");

    // 创建UI文本显示
    commands.spawn((
        Text::default(),
        Node {
            position_type: PositionType::Absolute,
            top: Val::Px(12.0),
            left: Val::Px(12.0),
            ..default()
        },
        children![
            TextSpan::new("点击切换IME。按回车开始新行。\\n\\n"),
            TextSpan::new("IME启用: "),
            TextSpan::new("false\\n"),
            TextSpan::new("IME激活: "),
            TextSpan::new("false\\n"),
            TextSpan::new("IME缓冲区: "),
            (
                TextSpan::new("\\n"),
                TextFont {
                    font: font.clone(),
                    ..default()
                },
            ),
        ],
    ));

    // 创建2D文本用于输入
    commands.spawn((
        Text2d::new(""),
        TextFont {
            font,
            font_size: 100.0,
            ..default()
        },
    ));
}

// 切换IME状态
fn toggle_ime(
    input: Res<ButtonInput<MouseButton>>,
    mut window: Single<&mut Window>,
    status_text: Single<Entity, (With<Node>, With<Text>)>,
    mut ui_writer: TextUiWriter,
) {
    if input.just_pressed(MouseButton::Left) {
        window.ime_position = window.cursor_position().unwrap();
        window.ime_enabled = !window.ime_enabled;

        *ui_writer.text(*status_text, 3) = format!("{}\\n", window.ime_enabled);
    }
}

// 监听IME事件
fn listen_ime_events(
    mut events: EventReader<Ime>,
    status_text: Single<Entity, (With<Node>, With<Text>)>,
    mut edit_text: Single<&mut Text2d, (Without<Node>, Without<Bubble>)>,
    mut ui_writer: TextUiWriter,
) {
    for event in events.read() {
        match event {
            Ime::Preedit { value, cursor } => {
                *ui_writer.text(*status_text, 5) = format!("{}\\n", value);
                *ui_writer.text(*status_text, 4) = "true\\n".to_string();
            }
            Ime::Commit { value } => {
                edit_text.0 = format!("{}{}", edit_text.0, value);
                *ui_writer.text(*status_text, 5) = "\\n".to_string();
                *ui_writer.text(*status_text, 4) = "false\\n".to_string();
            }
            Ime::Enabled { .. } => {
                *ui_writer.text(*status_text, 4) = "true\\n".to_string();
            }
            Ime::Disabled { .. } => {
                *ui_writer.text(*status_text, 4) = "false\\n".to_string();
            }
        }
    }
}

// 监听键盘输入事件
fn listen_keyboard_input_events(
    mut commands: Commands,
    mut events: EventReader<KeyboardInput>,
    edit_text: Single<(&mut Text2d, &TextFont), (Without<Node>, Without<Bubble>)>,
) {
    for event in events.read() {
        if !event.state.is_pressed() {
            continue;
        }

        match &event.logical_key {
            Key::Character(character) => {
                if is_printable_char(*character) {
                    edit_text.0 .0 = format!("{}{}", edit_text.0 .0, character);
                }
            }
            Key::Enter => {
                edit_text.0 .0 = format!("{}\\n", edit_text.0 .0);
            }
            Key::Backspace => {
                edit_text.0 .0.pop();
            }
            _ => {}
        }
    }
}

// 检查是否为可打印字符
fn is_printable_char(chr: char) -> bool {
    !chr.is_control() || chr == '\\n' || chr == '\\t'
}

```

**关键要点**:

- IME支持多语言输入
- 处理预编辑和提交事件
- 支持特殊键如回车、退格
- 需要合适的字体支持多语言字符

---

## 总结

Bevy的输入系统提供了全面的输入处理能力：

1. **键盘输入**: 基础按键、组合键、字符输入
2. **鼠标输入**: 按钮、移动、滚轮、光标控制
3. **触摸输入**: 多点触摸、手势识别
4. **游戏手柄**: 按钮、摇杆、扳机、震动
5. **文本输入**: IME支持、多语言输入

这些输入系统可以组合使用，创建丰富的交互体验。建议根据项目需求选择合适的输入方式。
