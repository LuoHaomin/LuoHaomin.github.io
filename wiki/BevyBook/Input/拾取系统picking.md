# 拾取系统（Picking）## 概述

**学习目标**：
- 理解 Bevy 拾取系统的基本概念
- 掌握网格拾取的使用
- 了解精灵拾取的使用
- 学会使用拾取事件

**前置知识要求**：
- Bevy 快速入门
- ECS 基础
- 输入处理基础
- 3D 开发基础（用于网格拾取）


## 核心概念

### 什么是拾取系统？

拾取系统是 Bevy 中用于检测鼠标或触摸输入是否与实体交互的功能。拾取系统可以用于实现点击、悬停、拖拽等交互功能。

**为什么需要拾取系统？**

1. **交互检测**：拾取系统可以检测鼠标或触摸输入是否与实体交互
2. **点击事件**：拾取系统可以触发点击事件
3. **悬停效果**：拾取系统可以实现悬停效果
4. **拖拽功能**：拾取系统可以实现拖拽功能

### 拾取系统的核心组件

Bevy 拾取系统包含以下核心组件：

- **Pickable**：可拾取组件，标记实体为可拾取
- **MeshPickingPlugin**：网格拾取插件，用于 3D 网格拾取
- **Pointer**：指针事件，用于处理鼠标和触摸输入
- **PointerInteraction**：指针交互，用于处理交互状态

## 基础用法

### 简单拾取

使用拾取系统实现简单的点击和拖拽功能。

**源代码文件**：`bevy/examples/picking/simple_picking.rs`

**代码示例**：

```rust
use bevy::prelude::*;

fn main() {
    App::new()
        .add_plugins((DefaultPlugins, MeshPickingPlugin))
        .add_systems(Startup, setup_scene)
        .run();
}

fn setup_scene(
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<StandardMaterial>>,
) {
    commands
        .spawn((
            Text::new("Click Me to get a box\nDrag cubes to rotate"),
            Node {
                position_type: PositionType::Absolute,
                top: percent(12),
                left: percent(12),
                ..default()
            },
        ))
        .observe(on_click_spawn_cube)
        .observe(|out: On<Pointer<Out>>, mut texts: Query<&mut TextColor>| {
            let mut text_color = texts.get_mut(out.entity).unwrap();
            text_color.0 = Color::WHITE;
        })
        .observe(
            |over: On<Pointer<Over>>, mut texts: Query<&mut TextColor>| {
                let mut color = texts.get_mut(over.entity).unwrap();
                color.0 = bevy::color::palettes::tailwind::CYAN_400.into();
            },
        );

    // 基础
    commands.spawn((
        Mesh3d(meshes.add(Circle::new(4.0))),
        MeshMaterial3d(materials.add(Color::WHITE)),
        Transform::from_rotation(Quat::from_rotation_x(-std::f32::consts::FRAC_PI_2)),
    ));

    // 光源
    commands.spawn((
        PointLight {
            shadows_enabled: true,
            ..default()
        },
        Transform::from_xyz(4.0, 8.0, 4.0),
    ));

    // 相机
    commands.spawn((
        Camera3d::default(),
        Transform::from_xyz(-2.5, 4.5, 9.0).looking_at(Vec3::ZERO, Vec3::Y),
    ));
}

fn on_click_spawn_cube(
    _click: On<Pointer<Click>>,
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<StandardMaterial>>,
    mut num: Local<usize>,
) {
    commands
        .spawn((
            Mesh3d(meshes.add(Cuboid::new(0.5, 0.5, 0.5))),
            MeshMaterial3d(materials.add(Color::srgb_u8(124, 144, 255))),
            Transform::from_xyz(0.0, 0.25 + 0.55 * *num as f32, 0.0),
        ))
        // 添加 MeshPickingPlugin 后，您可以向网格添加指针事件观察者：
        .observe(on_drag_rotate);
    *num += 1;
}

fn on_drag_rotate(drag: On<Pointer<Drag>>, mut transforms: Query<&mut Transform>) {
    if let Ok(mut transform) = transforms.get_mut(drag.entity) {
        transform.rotate_y(drag.delta.x * 0.02);
        transform.rotate_x(drag.delta.y * 0.02);
    }
}
```

**关键要点**：
- 使用 `MeshPickingPlugin` 启用网格拾取
- 使用 `observe()` 注册指针事件观察者
- 使用 `On<Pointer<Click>>` 处理点击事件
- 使用 `On<Pointer<Drag>>` 处理拖拽事件
- 使用 `On<Pointer<Over>>` 处理悬停事件
- 使用 `On<Pointer<Out>>` 处理离开事件

**说明**：
简单拾取是拾取系统的基础。通过使用简单拾取，可以实现点击和拖拽功能。

### 网格拾取

使用网格拾取实现 3D 网格的交互。

**源代码文件**：`bevy/examples/picking/mesh_picking.rs`

**代码示例**：

```rust
use bevy::{color::palettes::tailwind::*, picking::pointer::PointerInteraction, prelude::*};

fn main() {
    App::new()
        // MeshPickingPlugin 不是默认插件
        .add_plugins((DefaultPlugins, MeshPickingPlugin))
        .add_systems(Startup, setup_scene)
        .add_systems(Update, (draw_mesh_intersections, rotate))
        .run();
}

/// 一个标记组件，用于我们的形状，以便我们可以将它们与地平面分开查询。
#[derive(Component)]
struct Shape;

fn setup_scene(
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<StandardMaterial>>,
) {
    // 设置材质。
    let white_matl = materials.add(Color::WHITE);
    let ground_matl = materials.add(Color::from(GRAY_300));
    let hover_matl = materials.add(Color::from(CYAN_300));
    let pressed_matl = materials.add(Color::from(YELLOW_300));

    let shapes = [
        meshes.add(Cuboid::default()),
        meshes.add(Tetrahedron::default()),
        meshes.add(Capsule3d::default()),
        meshes.add(Torus::default()),
        meshes.add(Cylinder::default()),
        meshes.add(Cone::default()),
        meshes.add(ConicalFrustum::default()),
        meshes.add(Sphere::default().mesh().ico(5).unwrap()),
        meshes.add(Sphere::default().mesh().uv(32, 18)),
    ];

    // 生成形状。默认情况下，网格是可拾取的。
    for (i, shape) in shapes.into_iter().enumerate() {
        commands
            .spawn((
                Mesh3d(shape),
                MeshMaterial3d(white_matl.clone()),
                Transform::from_xyz(
                    -SHAPES_X_EXTENT / 2. + i as f32 / (num_shapes - 1) as f32 * SHAPES_X_EXTENT,
                    2.0,
                    Z_EXTENT / 2.,
                )
                .with_rotation(Quat::from_rotation_x(-PI / 4.)),
                Shape,
            ))
            .observe(update_material_on::<Pointer<Over>>(hover_matl.clone()))
            .observe(update_material_on::<Pointer<Out>>(white_matl.clone()))
            .observe(update_material_on::<Pointer<Press>>(pressed_matl.clone()))
            .observe(update_material_on::<Pointer<Release>>(hover_matl.clone()))
            .observe(rotate_on_drag);
    }
}
```

**关键要点**：
- 使用 `MeshPickingPlugin` 启用网格拾取
- 使用 `observe()` 注册指针事件观察者
- 使用 `On<Pointer<Over>>` 处理悬停事件
- 使用 `On<Pointer<Press>>` 处理按下事件
- 使用 `On<Pointer<Release>>` 处理释放事件

**说明**：
网格拾取是拾取系统的重要功能。通过使用网格拾取，可以实现 3D 网格的交互，如悬停效果和点击事件。

### 精灵拾取

使用精灵拾取实现 2D 精灵的交互。

**源代码文件**：`bevy/examples/picking/sprite_picking.rs`

**代码示例**：

```rust
use bevy::{prelude::*, sprite::Anchor};

fn main() {
    App::new()
        .add_plugins(DefaultPlugins.set(ImagePlugin::default_nearest()))
        .add_systems(Startup, (setup, setup_atlas))
        .add_systems(Update, (move_sprite, animate_sprite))
        .run();
}

fn setup(mut commands: Commands, asset_server: Res<AssetServer>) {
    commands.spawn(Camera2d);

    let len = 128.0;
    let sprite_size = Vec2::splat(len / 2.0);

    commands
        .spawn((Transform::default(), Visibility::default()))
        .with_children(|commands| {
            for (anchor_index, anchor) in [
                Anchor::TOP_LEFT,
                Anchor::TOP_CENTER,
                Anchor::TOP_RIGHT,
                Anchor::CENTER_LEFT,
                Anchor::CENTER,
                Anchor::CENTER_RIGHT,
                Anchor::BOTTOM_LEFT,
                Anchor::BOTTOM_CENTER,
                Anchor::BOTTOM_RIGHT,
            ]
            .iter()
            .enumerate()
            {
                let i = (anchor_index % 3) as f32;
                let j = (anchor_index / 3) as f32;

                // 在精灵后面生成黑色方块以显示锚点
                commands
                    .spawn((
                        Sprite::from_color(Color::BLACK, sprite_size),
                        Transform::from_xyz(i * len - len, j * len - len, -1.0),
                        Pickable::default(),
                    ))
                    .observe(recolor_on::<Pointer<Over>>(Color::srgb(0.0, 1.0, 1.0)))
                    .observe(recolor_on::<Pointer<Out>>(Color::BLACK))
                    .observe(recolor_on::<Pointer<Press>>(Color::srgb(1.0, 1.0, 0.0)))
                    .observe(recolor_on::<Pointer<Release>>(Color::srgb(0.0, 1.0, 1.0)));

                commands
                    .spawn((
                        Sprite {
                            image: asset_server.load("branding/bevy_bird_dark.png"),
                            custom_size: Some(sprite_size),
                            color: Color::srgb(1.0, 0.0, 0.0),
                            ..default()
                        },
                        anchor.to_owned(),
                        // 通过更改变换创建 3x3 锚点示例网格
                        Transform::from_xyz(i * len - len, j * len - len, 0.0)
                            .with_scale(Vec3::splat(1.0 + (i - 1.0) * 0.2))
                            .with_rotation(Quat::from_rotation_z((j - 1.0) * 0.2)),
                        Pickable::default(),
                    ))
                    .observe(recolor_on::<Pointer<Over>>(Color::srgb(0.0, 1.0, 0.0)))
                    .observe(recolor_on::<Pointer<Out>>(Color::srgb(1.0, 0.0, 0.0)))
                    .observe(recolor_on::<Pointer<Press>>(Color::srgb(0.0, 0.0, 1.0)))
                    .observe(recolor_on::<Pointer<Release>>(Color::srgb(0.0, 1.0, 0.0)));
            }
        });
}
```

**关键要点**：
- 使用 `Pickable` 组件标记实体为可拾取
- 使用 `observe()` 注册指针事件观察者
- 使用 `On<Pointer<Over>>` 处理悬停事件
- 使用 `On<Pointer<Press>>` 处理按下事件
- 使用 `On<Pointer<Release>>` 处理释放事件

**说明**：
精灵拾取是拾取系统的重要功能。通过使用精灵拾取，可以实现 2D 精灵的交互，如悬停效果和点击事件。

## 进阶用法

### 拾取事件

使用拾取事件处理复杂的交互。

**关键信息**：
- 使用 `On<Pointer<Click>>` 处理点击事件
- 使用 `On<Pointer<Drag>>` 处理拖拽事件
- 使用 `On<Pointer<Over>>` 处理悬停事件
- 使用 `On<Pointer<Out>>` 处理离开事件
- 使用 `On<Pointer<Press>>` 处理按下事件
- 使用 `On<Pointer<Release>>` 处理释放事件

**说明**：
拾取事件是拾取系统的重要功能。通过使用拾取事件，可以实现复杂的交互，如点击、拖拽、悬停等。

### 拾取调试

使用拾取调试工具调试拾取问题。

**源代码文件**：`bevy/examples/picking/debug_picking.rs`

**关键信息**：
- 使用 `DebugPickingPlugin` 启用拾取调试
- 使用 `DebugPickingMode` 控制调试模式
- 使用 `DebugPickingBackend` 控制调试后端

**说明**：
拾取调试是拾取系统的重要功能。通过使用拾取调试，可以调试拾取问题，了解拾取系统的行为。

## 实际应用

### 在游戏开发中的应用场景

拾取系统在游戏开发中有广泛的应用：

1. **点击交互**：实现点击实体触发事件
2. **悬停效果**：实现鼠标悬停时的视觉效果
3. **拖拽功能**：实现拖拽实体移动
4. **选择功能**：实现选择实体的功能
5. **UI 交互**：实现 UI 元素的交互

### 常见问题

**问题 1**：如何启用拾取系统？

**解决方案**：
- 使用 `MeshPickingPlugin` 启用网格拾取
- 使用 `Pickable` 组件标记实体为可拾取
- 使用 `observe()` 注册指针事件观察者

**问题 2**：如何处理拾取事件？

**解决方案**：
- 使用 `On<Pointer<Click>>` 处理点击事件
- 使用 `On<Pointer<Drag>>` 处理拖拽事件
- 使用 `On<Pointer<Over>>` 处理悬停事件
- 使用 `On<Pointer<Out>>` 处理离开事件

**问题 3**：如何实现悬停效果？

**解决方案**：
- 使用 `On<Pointer<Over>>` 处理悬停事件
- 使用 `On<Pointer<Out>>` 处理离开事件
- 在事件处理中更新实体的材质或颜色

### 性能考虑

1. **拾取插件**：拾取插件是高效的，可以频繁使用
2. **拾取事件**：拾取事件处理是高效的，可以大量使用
3. **拾取调试**：拾取调试应仅在开发时使用，避免影响性能

## 相关资源

**相关源代码文件**：
- `bevy/examples/picking/simple_picking.rs` - 简单拾取示例
- `bevy/examples/picking/mesh_picking.rs` - 网格拾取示例
- `bevy/examples/picking/sprite_picking.rs` - 精灵拾取示例
- `bevy/examples/picking/debug_picking.rs` - 拾取调试示例

**官方文档链接**：
- [Bevy 拾取系统](https://docs.rs/bevy/latest/bevy/picking/index.html)
- [Bevy 拾取示例](https://github.com/bevyengine/bevy/tree/main/examples/picking)

**进一步学习建议**：
- 学习输入处理，了解输入系统
- 学习 3D 开发，了解 3D 渲染基础
- 学习 UI 系统，了解 UI 交互

---

**索引**：[返回上级目录](/wiki/BevyBook/Input/)
