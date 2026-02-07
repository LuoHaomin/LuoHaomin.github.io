# 相机系统（Camera）## 概述

**学习目标**：
- 理解 Bevy 相机系统的基本概念
- 掌握相机控制器的使用
- 了解相机投影的使用
- 学会使用相机轨道和第一人称相机

**前置知识要求**：
- Bevy 快速入门
- ECS 基础
- 3D 开发基础
- 输入处理基础


## 核心概念

### 什么是相机系统？

相机系统是 Bevy 中用于控制视角的功能。相机系统提供了多种相机类型和控制方式，包括轨道相机、第一人称相机、自定义投影等。

**为什么需要相机系统？**

1. **视角控制**：相机系统可以控制游戏的视角
2. **相机移动**：相机系统可以实现相机的移动和旋转
3. **相机投影**：相机系统可以自定义相机的投影方式
4. **相机效果**：相机系统可以实现相机特效（如屏幕抖动）

### 相机系统的核心组件

Bevy 相机系统包含以下核心组件：

- **Camera3d**：3D 相机组件
- **Camera2d**：2D 相机组件
- **Projection**：投影组件
- **Transform**：变换组件，用于控制相机位置和旋转

## 基础用法

### 相机控制器

使用相机控制器实现自由相机。

**源代码文件**：`bevy/examples/helpers/camera_controller.rs`

**代码示例**：

```rust
use bevy::{
    input::mouse::{AccumulatedMouseMotion, AccumulatedMouseScroll, MouseScrollUnit},
    prelude::*,
    window::{CursorGrabMode, CursorOptions},
};
use std::{f32::consts::*, fmt};

/// 自由相机风格的相机控制器插件。
pub struct CameraControllerPlugin;

impl Plugin for CameraControllerPlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(Update, run_camera_controller);
    }
}

/// 相机控制器组件。
#[derive(Component)]
pub struct CameraController {
    /// 当为 `true` 时启用此 [`CameraController`]。
    pub enabled: bool,
    /// 指示此控制器是否已由 [`CameraControllerPlugin`] 初始化。
    pub initialized: bool,
    /// 俯仰和偏航旋转速度的乘数。
    pub sensitivity: f32,
    /// 向前平移的 [`KeyCode`]。
    pub key_forward: KeyCode,
    /// 向后平移的 [`KeyCode`]。
    pub key_back: KeyCode,
    /// 向左平移的 [`KeyCode`]。
    pub key_left: KeyCode,
    /// 向右平移的 [`KeyCode`]。
    pub key_right: KeyCode,
    /// 向上平移的 [`KeyCode`]。
    pub key_up: KeyCode,
    /// 向下平移的 [`KeyCode`]。
    pub key_down: KeyCode,
    /// 使用 [`run_speed`](CameraController::run_speed) 而不是
    /// [`walk_speed`](CameraController::walk_speed) 进行平移的 [`KeyCode`]。
    pub key_run: KeyCode,
    /// 用于抓取鼠标焦点的 [`MouseButton`]。
    pub mouse_key_cursor_grab: MouseButton,
    /// 用于抓取键盘焦点的 [`KeyCode`]。
    pub keyboard_key_toggle_cursor_grab: KeyCode,
    /// 未修改平移速度的乘数。
    pub walk_speed: f32,
    /// 运行平移速度的乘数。
    pub run_speed: f32,
    /// 鼠标滚轮修改 [`walk_speed`](CameraController::walk_speed)
    /// 和 [`run_speed`](CameraController::run_speed) 的乘数。
    pub scroll_factor: f32,
    /// 用于随时间指数衰减 [`velocity`](CameraController::velocity) 的摩擦因子。
    pub friction: f32,
    /// 此 [`CameraController`] 的俯仰旋转。
    pub pitch: f32,
    /// 此 [`CameraController`] 的偏航旋转。
    pub yaw: f32,
    /// 此 [`CameraController`] 的平移速度。
    pub velocity: Vec3,
}

impl Default for CameraController {
    fn default() -> Self {
        Self {
            enabled: true,
            initialized: false,
            sensitivity: 1.0,
            key_forward: KeyCode::KeyW,
            key_back: KeyCode::KeyS,
            key_left: KeyCode::KeyA,
            key_right: KeyCode::KeyD,
            key_up: KeyCode::KeyE,
            key_down: KeyCode::KeyQ,
            key_run: KeyCode::ShiftLeft,
            mouse_key_cursor_grab: MouseButton::Left,
            keyboard_key_toggle_cursor_grab: KeyCode::KeyM,
            walk_speed: 5.0,
            run_speed: 15.0,
            scroll_factor: 0.1,
            friction: 0.5,
            pitch: 0.0,
            yaw: 0.0,
            velocity: Vec3::ZERO,
        }
    }
}
```

**关键要点**：
- 使用 `CameraController` 组件控制相机
- 使用 `CameraControllerPlugin` 插件添加相机控制器系统
- 使用键盘和鼠标控制相机移动和旋转
- 使用 `CursorGrabMode` 控制鼠标抓取模式

**说明**：
相机控制器是相机系统的基础。通过使用相机控制器，可以实现自由相机，控制游戏的视角。

### 相机轨道

使用相机轨道实现轨道相机。

**源代码文件**：`bevy/examples/camera/camera_orbit.rs`

**代码示例**：

```rust
use std::{f32::consts::FRAC_PI_2, ops::Range};
use bevy::{input::mouse::AccumulatedMouseMotion, prelude::*};

#[derive(Debug, Resource)]
struct CameraSettings {
    pub orbit_distance: f32,
    pub pitch_speed: f32,
    // 将俯仰限制在此范围内
    pub pitch_range: Range<f32>,
    pub roll_speed: f32,
    pub yaw_speed: f32,
}

impl Default for CameraSettings {
    fn default() -> Self {
        // 限制俯仰可以防止超过 90° 向上或向下的意外旋转。
        let pitch_limit = FRAC_PI_2 - 0.01;
        Self {
            // 这些值完全是任意的，选择它们是因为它们似乎为这个示例产生"合理"的结果。
            // 根据需要调整。
            orbit_distance: 20.0,
            pitch_speed: 0.003,
            pitch_range: -pitch_limit..pitch_limit,
            roll_speed: 1.0,
            yaw_speed: 0.004,
        }
    }
}

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .init_resource::<CameraSettings>()
        .add_systems(Startup, (setup, instructions))
        .add_systems(Update, orbit)
        .run();
}

fn orbit(
    mut camera: Single<&mut Transform, With<Camera>>,
    settings: Res<CameraSettings>,
    mouse_motion: Res<AccumulatedMouseMotion>,
    mouse_buttons: Res<ButtonInput<MouseButton>>,
) {
    // 根据鼠标移动更新俯仰和偏航
    let mut pitch_delta = 0.0;
    let mut yaw_delta = 0.0;
    let mut roll_delta = 0.0;

    if mouse_motion.is_changed() {
        let delta = mouse_motion.delta();
        pitch_delta -= delta.y * settings.pitch_speed;
        yaw_delta -= delta.x * settings.yaw_speed;
    }

    // 根据鼠标按钮更新滚动
    if mouse_buttons.pressed(MouseButton::Left) {
        roll_delta += settings.roll_speed;
    }
    if mouse_buttons.pressed(MouseButton::Right) {
        roll_delta -= settings.roll_speed;
    }

    // 更新相机变换
    let mut pitch = camera.rotation.to_euler(EulerRot::YXZ).1;
    let mut yaw = camera.rotation.to_euler(EulerRot::YXZ).0;
    let mut roll = camera.rotation.to_euler(EulerRot::YXZ).2;

    pitch += pitch_delta;
    pitch = pitch.clamp(settings.pitch_range.start, settings.pitch_range.end);
    yaw += yaw_delta;
    roll += roll_delta;

    // 计算相机位置
    let rotation = Quat::from_euler(EulerRot::YXZ, yaw, pitch, roll);
    let position = rotation * Vec3::new(0.0, 0.0, settings.orbit_distance);

    camera.translation = position;
    camera.rotation = rotation;
}
```

**关键要点**：
- 使用 `AccumulatedMouseMotion` 获取鼠标移动
- 使用 `ButtonInput<MouseButton>` 获取鼠标按钮状态
- 使用 `Transform` 控制相机位置和旋转
- 使用 `Quat::from_euler` 创建旋转四元数

**说明**：
相机轨道是相机系统的重要功能。通过使用相机轨道，可以实现轨道相机，围绕场景旋转相机。

### 自定义投影

使用自定义投影实现特殊投影效果。

**源代码文件**：`bevy/examples/camera/custom_projection.rs`

**代码示例**：

```rust
use bevy::camera::CameraProjection;
use bevy::prelude::*;

/// 类似于透视投影，但消失点不居中。
#[derive(Debug, Clone)]
struct ObliquePerspectiveProjection {
    horizontal_obliqueness: f32,
    vertical_obliqueness: f32,
    perspective: PerspectiveProjection,
}

/// 为我们的自定义投影实现 [`CameraProjection`] trait：
impl CameraProjection for ObliquePerspectiveProjection {
    fn get_clip_from_view(&self) -> Mat4 {
        let mut mat = self.perspective.get_clip_from_view();
        mat.col_mut(2)[0] = self.horizontal_obliqueness;
        mat.col_mut(2)[1] = self.vertical_obliqueness;
        mat
    }

    fn get_clip_from_view_for_sub(&self, sub_view: &bevy::camera::SubCameraView) -> Mat4 {
        let mut mat = self.perspective.get_clip_from_view_for_sub(sub_view);
        mat.col_mut(2)[0] = self.horizontal_obliqueness;
        mat.col_mut(2)[1] = self.vertical_obliqueness;
        mat
    }

    fn update(&mut self, width: f32, height: f32) {
        self.perspective.update(width, height);
    }

    fn far(&self) -> f32 {
        self.perspective.far
    }

    fn get_frustum_corners(&self, z_near: f32, z_far: f32) -> [Vec3A; 8] {
        self.perspective.get_frustum_corners(z_near, z_far)
    }
}

fn setup(
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<StandardMaterial>>,
) {
    commands.spawn((
        Camera3d::default(),
        // 使用我们的自定义投影：
        Projection::custom(ObliquePerspectiveProjection {
            horizontal_obliqueness: 0.2,
            vertical_obliqueness: 0.6,
            perspective: PerspectiveProjection::default(),
        }),
        Transform::from_xyz(-2.5, 4.5, 9.0).looking_at(Vec3::ZERO, Vec3::Y),
    ));
}
```

**关键要点**：
- 使用 `CameraProjection` trait 实现自定义投影
- 使用 `Projection::custom()` 设置自定义投影
- 使用 `get_clip_from_view()` 获取投影矩阵
- 使用 `update()` 更新投影参数

**说明**：
自定义投影是相机系统的高级功能。通过使用自定义投影，可以实现特殊的投影效果，如斜投影。

## 进阶用法

### 屏幕抖动

使用屏幕抖动实现相机特效。

**源代码文件**：`bevy/examples/camera/2d_screen_shake.rs`

**关键信息**：
- 使用 `Transform` 控制相机位置
- 使用随机数生成抖动效果
- 使用时间控制抖动持续时间
- 使用衰减函数平滑抖动

**说明**：
屏幕抖动是相机系统的重要功能。通过使用屏幕抖动，可以实现相机特效，增强游戏体验。

### 第一人称相机

使用第一人称相机实现第一人称视角。

**源代码文件**：`bevy/examples/camera/first_person_view_model.rs`

**关键信息**：
- 使用 `CameraController` 控制相机
- 使用鼠标控制视角
- 使用键盘控制移动
- 使用 `CursorGrabMode` 控制鼠标抓取

**说明**：
第一人称相机是相机系统的重要功能。通过使用第一人称相机，可以实现第一人称视角，增强游戏沉浸感。

## 实际应用

### 在游戏开发中的应用场景

相机系统在游戏开发中有广泛的应用：

1. **视角控制**：控制游戏的视角
2. **相机移动**：实现相机的移动和旋转
3. **相机投影**：自定义相机的投影方式
4. **相机效果**：实现相机特效（如屏幕抖动）
5. **相机切换**：实现不同相机之间的切换

### 常见问题

**问题 1**：如何控制相机移动？

**解决方案**：
- 使用 `CameraController` 组件控制相机
- 使用键盘和鼠标控制相机移动和旋转
- 使用 `Transform` 控制相机位置和旋转

**问题 2**：如何实现相机轨道？

**解决方案**：
- 使用 `AccumulatedMouseMotion` 获取鼠标移动
- 使用 `Transform` 控制相机位置和旋转
- 使用 `Quat::from_euler` 创建旋转四元数

**问题 3**：如何自定义相机投影？

**解决方案**：
- 使用 `CameraProjection` trait 实现自定义投影
- 使用 `Projection::custom()` 设置自定义投影
- 使用 `get_clip_from_view()` 获取投影矩阵

### 性能考虑

1. **相机控制器**：相机控制器更新是高效的，可以频繁使用
2. **相机投影**：自定义投影计算可能较慢，应谨慎使用
3. **相机效果**：相机效果应适度使用，避免影响性能

## 相关资源

**相关源代码文件**：
- `bevy/examples/helpers/camera_controller.rs` - 相机控制器示例
- `bevy/examples/camera/camera_orbit.rs` - 相机轨道示例
- `bevy/examples/camera/custom_projection.rs` - 自定义投影示例
- `bevy/examples/camera/2d_screen_shake.rs` - 屏幕抖动示例
- `bevy/examples/camera/first_person_view_model.rs` - 第一人称相机示例

**官方文档链接**：
- [Bevy 相机系统](https://docs.rs/bevy/latest/bevy/camera/index.html)
- [Bevy 相机示例](https://github.com/bevyengine/bevy/tree/main/examples/camera)

**进一步学习建议**：
- 学习 3D 开发，了解 3D 渲染基础
- 学习输入处理，了解输入系统

---

**索引**：[返回上级目录](/wiki/BevyBook/3D_Graphics/)
