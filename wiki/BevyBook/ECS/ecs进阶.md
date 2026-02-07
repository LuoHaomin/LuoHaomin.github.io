# ECS 进阶## 概述

**学习目标**：
- 掌握变更检测（Change Detection）的使用方法
- 理解组件生命周期钩子（Component Hooks）的应用场景
- 学会使用关系系统（Relationships）建立实体间的关系
- 掌握并行查询（Parallel Queries）的性能优化技巧
- 理解查询组合（Query Combinations）的使用方法
- 学会使用观察者模式（Observers）响应组件变化
- 了解消息系统、错误处理、动态 ECS 等高级功能

**前置知识要求**：
- 核心编程框架（ECS）
- 组件（Components）
- 实体（Entities）
- 系统（Systems）
- 查询（Queries）
- 资源（Resources）
- 系统调度（Schedule & App）


## 核心概念

### 为什么需要高级 ECS 功能？

在复杂的游戏开发中，我们需要：

1. **性能优化**：通过并行查询提高系统执行效率
2. **变更响应**：检测组件和资源的变化，及时响应
3. **关系管理**：建立和维护实体间的复杂关系
4. **生命周期管理**：在组件的生命周期关键点执行逻辑
5. **事件驱动**：使用消息系统和观察者模式实现事件驱动逻辑

### ECS 进阶功能概览

- **变更检测**：检测组件和资源的变化
- **组件生命周期钩子**：在组件添加、插入、替换、移除时执行逻辑
- **关系系统**：建立自定义的实体关系
- **并行查询**：使用并行迭代器提高性能
- **查询组合**：处理实体间的交互
- **观察者模式**：响应组件生命周期事件和自定义事件
- **消息系统**：使用消息实现系统间通信
- **错误处理**：处理系统执行中的错误
- **动态 ECS**：动态创建组件和实体

## 基础用法

### 变更检测（Change Detection）

变更检测用于检测组件和资源的变化，是响应式编程的基础。

**源代码文件**：`bevy/examples/ecs/change_detection.rs`

**代码示例**：

```rust
use bevy::prelude::*;

#[derive(Component, PartialEq, Debug)]
struct MyComponent(f32);

#[derive(Resource, PartialEq, Debug)]
struct MyResource(f32);

fn change_component(time: Res<Time>, mut query: Query<(Entity, &mut MyComponent)>) {
    for (entity, mut component) in &mut query {
        if rand::rng().random_bool(0.1) {
            let new_component = MyComponent(time.elapsed_secs().round());
            info!("New value: {new_component:?} {entity}");
            // 变更检测发生在可变解引用时，不考虑值是否实际相等
            // 为了避免在值未实际改变时触发变更检测，可以使用 `set_if_neq` 方法
            // 该方法要求组件实现 PartialEq
            component.set_if_neq(new_component);
        }
    }
}

fn change_detection(
    changed_components: Query<Ref<MyComponent>, Changed<MyComponent>>,
    my_resource: Res<MyResource>,
) {
    for component in &changed_components {
        // 默认情况下，你只能知道组件被改变了
        // 但如果有多个系统修改同一个组件，如何知道是哪个系统导致的改变？
        warn!(
            "Change detected!\n\t-> value: {:?}\n\t-> added: {}\n\t-> changed: {}\n\t-> changed by: {}",
            component,
            component.is_added(),
            component.is_changed(),
            // 如果启用 `track_location` 特性，可以解锁 `changed_by()` 方法
            // 它返回组件或资源被改变的文件和行号
            // 不建议在发布的游戏中使用，但对调试很有用！
            component.changed_by()
        );
    }

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
- 使用 `Changed<T>` 过滤器查询已变更的组件
- 使用 `Added<T>` 过滤器查询新添加的组件
- 使用 `Ref<T>` 系统参数访问变更检测信息，但不过滤查询
- 使用 `set_if_neq()` 方法避免不必要的变更检测
- 使用 `is_changed()` 和 `is_added()` 检查变更状态
- 使用 `changed_by()` 方法（需要 `track_location` 特性）获取变更位置

**说明**：
变更检测是 Bevy ECS 的核心功能之一。当组件或资源被修改时，Bevy 会自动跟踪这些变更。使用 `Changed<T>` 过滤器可以只查询已变更的组件，这对于性能优化很重要。`set_if_neq()` 方法可以避免在值未实际改变时触发变更检测，这对于实现 `PartialEq` 的组件很有用。

### 组件生命周期钩子（Component Hooks）

组件生命周期钩子允许在组件的生命周期关键点执行逻辑。

**源代码文件**：`bevy/examples/ecs/component_hooks.rs`

**代码示例**：

```rust
use bevy::{
    ecs::component::{Mutable, StorageType},
    ecs::lifecycle::{ComponentHook, HookContext},
    prelude::*,
};
use std::collections::HashMap;

#[derive(Debug)]
struct MyComponent(KeyCode);

impl Component for MyComponent {
    const STORAGE_TYPE: StorageType = StorageType::Table;
    type Mutability = Mutable;

    fn on_add() -> Option<ComponentHook> {
        // 如果没有 on_add 钩子，返回 None
        // 注意这是不实现钩子时的默认行为
        None
    }
}

fn setup(world: &mut World) {
    // 为了注册组件钩子，组件必须：
    // - 当前未被世界中的任何实体使用
    // - 尚未注册该类型的钩子
    // 这是为了防止覆盖插件和其他 crate 中定义的钩子，并保持性能
    world
        .register_component_hooks::<MyComponent>()
        // 有 4 种组件生命周期钩子：`on_add`、`on_insert`、`on_replace` 和 `on_remove`
        // 钩子有 2 个参数：
        // - 一个 `DeferredWorld`，允许访问资源和组件数据以及 `Commands`
        // - 一个 `HookContext`，提供以下上下文信息：
        //   - 触发钩子的实体
        //   - 触发组件的组件 ID，主要用于动态组件
        //   - 导致钩子触发的代码位置
        //
        // `on_add` 将在组件插入到没有该组件的实体上时触发
        .on_add(
            |mut world,
             HookContext {
                 entity,
                 component_id,
                 caller,
                 ..
             }| {
                // 可以在钩子内访问组件数据
                let value = world.get::<MyComponent>(entity).unwrap().0;
                println!(
                    "{component_id:?} added to {entity} with value {value:?}{}",
                    caller
                        .map(|location| format!("due to {location}"))
                        .unwrap_or_default()
                );
                // 或访问资源
                world
                    .resource_mut::<MyComponentIndex>()
                    .insert(value, entity);
            },
        )
        // `on_insert` 将在组件插入到实体上时触发，无论实体是否已有该组件
        // 如果 `on_add` 运行了，则在 `on_add` 之后运行
        .on_insert(|world, _| {
            println!("Current Index: {:?}", world.resource::<MyComponentIndex>());
        })
        // `on_replace` 将在组件插入到已有该组件的实体上时触发
        // 在值被替换之前运行
        // 当组件从实体移除时也会触发，在 `on_remove` 之前运行
        .on_replace(|mut world, context| {
            let value = world.get::<MyComponent>(context.entity).unwrap().0;
            world.resource_mut::<MyComponentIndex>().remove(&value);
        })
        // `on_remove` 将在组件从实体移除时触发
        // 由于它在组件移除之前运行，你仍然可以访问组件数据
        .on_remove(
            |mut world,
             HookContext {
                 entity,
                 component_id,
                 caller,
                 ..
             }| {
                let value = world.get::<MyComponent>(entity).unwrap().0;
                println!(
                    "{component_id:?} removed from {entity} with value {value:?}{}",
                    caller
                        .map(|location| format!("due to {location}"))
                        .unwrap_or_default()
                );
                // 也可以通过 `.commands()` 发出命令
                world.commands().entity(entity).despawn();
            },
        );
}
```

**关键要点**：
- 有 4 种组件生命周期钩子：`on_add`、`on_insert`、`on_replace`、`on_remove`
- 钩子可以访问 `DeferredWorld` 和 `HookContext`
- 钩子可以访问组件数据、资源和 `Commands`
- 钩子可以发送消息
- 组件必须未被使用且未注册钩子才能注册钩子

**说明**：
组件生命周期钩子用于在组件的生命周期关键点执行逻辑。它们对于维护索引、强制执行结构规则等场景很有用。但要注意，尽可能使用 Bevy 的变更检测或事件来响应组件变化，因为事件通常提供更好的性能和更灵活的集成。

### 关系系统（Relationships）

关系系统允许建立自定义的实体关系，类似于内置的 `ChildOf`/`Children` 关系。

**源代码文件**：`bevy/examples/ecs/relationships.rs`

**代码示例**：

```rust
use bevy::prelude::*;

/// 此实体正在瞄准的实体
///
/// 这是关系的真实来源，可以直接修改以改变目标
#[derive(Component, Debug)]
#[relationship(relationship_target = TargetedBy)]
struct Targeting(Entity);

/// 所有正在瞄准此实体的实体
///
/// 此组件使用派生 [`Relationship`] trait 引入的组件钩子进行响应式更新
/// 我们不应该直接修改此组件，但可以安全地读取其字段
#[derive(Component, Debug)]
#[relationship_target(relationship = Targeting)]
struct TargetedBy(Vec<Entity>);

fn spawning_entities_with_relationships(mut commands: Commands) {
    // 调用 `.id()` 在生成实体后将返回生成实体的 `Entity` 标识符
    // 即使实体本身尚未在世界中实例化
    let alice = commands.spawn(Name::new("Alice")).id();
    // 关系只是组件，所以我们可以将它们添加到正在生成的 bundle 中
    let bob = commands.spawn((Name::new("Bob"), Targeting(alice))).id();

    // `with_related` 和 `with_related_entities` 辅助方法可以更符合人体工程学地添加关系
    let charlie = commands
        .spawn((Name::new("Charlie"), Targeting(bob)))
        // `with_related` 方法将生成一个带有 `Targeting` 关系的 bundle
        .with_related::<Targeting>(Name::new("James"))
        // `with_related_entities` 方法将自动将 `Targeting` 组件添加到闭包内生成的任何实体
        .with_related_entities::<Targeting>(|related_spawner_commands| {
            // 我们可以在这里生成多个实体，它们都会瞄准 `charlie`
            related_spawner_commands.spawn(Name::new("Devon"));
        })
        .id();

    // 简单地插入 `Targeting` 组件将自动创建并更新目标实体上的 `TargetedBy` 组件
    // 我们可以在任何时候这样做；不仅仅是在实体生成时
    commands.entity(alice).insert(Targeting(charlie));
    }

fn mutate_relationships(name_query: Query<(Entity, &Name)>, mut commands: Commands) {
    // 关系组件是不可变的！我们不能可变地查询 `Targeting` 组件并直接修改它
    // 但我们可以插入一个新的 `Targeting` 组件来替换旧的
    // 这允许 `Targeting` 组件上的钩子正确更新 `TargetedBy` 组件
    // `TargetedBy` 组件将自动更新！
    let devon = name_query
        .iter()
        .find(|(_entity, name)| name.as_str() == "Devon")
        .unwrap()
        .0;

    let alice = name_query
        .iter()
        .find(|(_entity, name)| name.as_str() == "Alice")
        .unwrap()
        .0;

    println!("Making Devon target Alice.\n");
    commands.entity(devon).insert(Targeting(alice));
}
```

**关键要点**：
- 使用 `#[relationship(relationship_target = TargetedBy)]` 定义关系组件
- 使用 `#[relationship_target(relationship = Targeting)]` 定义关系目标组件
- 关系组件是真实来源，可以直接修改
- 关系目标组件是响应式更新的，不应直接修改
- 使用 `with_related` 和 `with_related_entities` 辅助方法添加关系
- 插入关系组件会自动更新关系目标组件

**说明**：
关系系统允许建立自定义的实体关系。Bevy 内置了 `ChildOf`/`Children` 关系用于变换和可见性传播，但你可以定义自己的关系。关系组件是真实来源，可以直接修改，而关系目标组件是响应式更新的，不应直接修改。

### 并行查询（Parallel Queries）

并行查询使用并行迭代器提高系统执行效率。

**源代码文件**：`bevy/examples/ecs/parallel_query.rs`

**代码示例**：

```rust
use bevy::{ecs::batching::BatchingStrategy, prelude::*};

#[derive(Component, Deref)]
struct Velocity(Vec2);

// 根据速度移动精灵
fn move_system(mut sprites: Query<(&mut Transform, &Velocity)>) {
    // 在 ComputeTaskPool 上并行计算每个精灵的新位置
    //
    // 此示例仅用于演示目的。对于像加法这样便宜的操作，在只有 128 个元素时
    // 使用 ParallelIterator 通常不会比使用普通 Iterator 更快
    // 有关何时使用或不使用 ParallelIterator 的更多信息，请参阅 ParallelIterator 文档
    sprites
        .par_iter_mut()
        .for_each(|(mut transform, velocity)| {
            transform.translation += velocity.extend(0.0);
        });
}

// 在窗口外反弹精灵
fn bounce_system(window: Query<&Window>, mut sprites: Query<(&Transform, &mut Velocity)>) {
    let Ok(window) = window.single() else {
        return;
    };
    let width = window.width();
    let height = window.height();
    // 也可以覆盖默认批次大小
    // 在这种情况下，选择批次大小为 32 以限制 ParallelIterator 的开销
    sprites
        .par_iter_mut()
        .batching_strategy(BatchingStrategy::fixed(32))
        .for_each(|(transform, mut v)| {
            // ...
        });
}
```

**关键要点**：
- 使用 `par_iter_mut()` 获取并行迭代器
- 使用 `batching_strategy()` 自定义批处理策略
- 并行查询适用于计算密集型操作
- 对于简单操作，并行查询可能不会更快

**注意事项**：
- 并行查询适用于计算密集型操作
- 对于简单操作，并行查询可能不会更快
- 可以使用 `batching_strategy()` 自定义批处理策略

**最佳实践**：
- 只在计算密集型操作时使用并行查询
- 对于简单操作，使用普通查询
- 根据操作复杂度调整批处理策略

### 查询组合（Query Combinations）

查询组合用于处理实体间的交互，如碰撞检测。

**源代码文件**：`bevy/examples/ecs/iter_combinations.rs`

**代码示例**：

```rust
use bevy::prelude::*;

const GRAVITY_CONSTANT: f32 = 0.001;

#[derive(Component, Default)]
struct Mass(f32);
#[derive(Component, Default)]
struct Acceleration(Vec3);

fn interact_bodies(mut query: Query<(&Mass, &GlobalTransform, &mut Acceleration)>) {
    let mut iter = query.iter_combinations_mut();
    while let Some([(Mass(m1), transform1, mut acc1), (Mass(m2), transform2, mut acc2)]) =
        iter.fetch_next()
    {
        let delta = transform2.translation() - transform1.translation();
        let distance_sq: f32 = delta.length_squared();

        let f = GRAVITY_CONSTANT / distance_sq;
        let force_unit_mass = delta * f;
        acc1.0 += force_unit_mass * *m2;
        acc2.0 -= force_unit_mass * *m1;
    }
}
```

**关键要点**：
- 使用 `iter_combinations_mut()` 获取可变组合迭代器
- 使用 `fetch_next()` 获取下一对实体
- 查询组合会跳过重复的组合（如 (A, B) 和 (B, A)）
- 查询组合用于处理实体间的成对交互

**注意事项**：
- 查询组合用于处理实体间的成对交互
- 查询组合会跳过重复的组合
- 查询组合对于大量实体可能较慢

**最佳实践**：
- 使用查询组合处理碰撞检测、物理交互等场景
- 注意性能影响，对于大量实体考虑使用空间分区
- 考虑使用并行查询组合提高性能

### 观察者模式（Observers）

观察者模式用于响应组件生命周期事件和自定义事件。

**源代码文件**：`bevy/examples/ecs/observers.rs`

**代码示例**：

```rust
use bevy::prelude::*;

#[derive(Component)]
struct Mine {
    pos: Vec2,
    size: f32,
}

/// 这是一个普通的 [`Event`]。任何观察它的观察者都会在它被触发时运行
#[derive(Event)]
struct ExplodeMines {
    pos: Vec2,
    radius: f32,
}

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
        .add_systems(Startup, setup)
        .add_systems(Update, (draw_shapes, handle_click))
        // 观察者是在事件被"触发"时运行的系统
        // 此观察者在 `ExplodeMines` 被触发时运行
        .add_observer(
            |explode_mines: On<ExplodeMines>,
             mines: Query<&Mine>,
             index: Res<SpatialIndex>,
             mut commands: Commands| {
                // 访问资源
                for entity in index.get_nearby(explode_mines.pos) {
                    // 运行查询
                    let mine = mines.get(entity).unwrap();
                    if mine.pos.distance(explode_mines.pos) < mine.size + explode_mines.radius {
                        // 并排队命令，包括触发其他事件
                        // 这里我们为实体 `e` 触发 `Explode` 事件
                        commands.trigger(Explode { entity });
                    }
                }
            },
        )
        // 此观察者在 `Mine` 组件添加到实体时运行，并将其放置在简单的空间索引中
        .add_observer(on_add_mine)
        // 此观察者在 `Mine` 组件从实体移除时运行（包括销毁它）
        // 并将其从空间索引中移除
        .add_observer(on_remove_mine)
        .run();
}

fn on_add_mine(add: On<Add, Mine>, query: Query<&Mine>, mut index: ResMut<SpatialIndex>) {
    let mine = query.get(add.entity).unwrap();
    let tile = (
        (mine.pos.x / CELL_SIZE).floor() as i32,
        (mine.pos.y / CELL_SIZE).floor() as i32,
    );
    index.map.entry(tile).or_default().insert(add.entity);
}

fn on_remove_mine(remove: On<Remove, Mine>, query: Query<&Mine>, mut index: ResMut<SpatialIndex>) {
    let mine = query.get(remove.entity).unwrap();
    let tile = (
        (mine.pos.x / CELL_SIZE).floor() as i32,
        (mine.pos.y / CELL_SIZE).floor() as i32,
    );
    index.map.entry(tile).and_modify(|set| {
        set.remove(&remove.entity);
    });
}
```

**关键要点**：
- 观察者用于响应组件生命周期事件和自定义事件
- 使用 `On<Add, T>` 响应组件添加
- 使用 `On<Remove, T>` 响应组件移除
- 使用 `On<Event>` 响应自定义事件
- 观察者可以访问资源、运行查询、发出命令

**注意事项**：
- 观察者用于响应组件生命周期事件和自定义事件
- 观察者可以访问资源、运行查询、发出命令
- 观察者比组件钩子更灵活，但开销更大

**最佳实践**：
- 使用观察者响应组件生命周期事件
- 使用观察者响应自定义事件
- 考虑性能影响，观察者可能比组件钩子更灵活但开销更大

## 进阶用法

### 消息系统（Messages）

消息系统用于实现系统间通信。

**源代码文件**：`bevy/examples/ecs/message.rs`

**代码示例**：

```rust
use bevy::prelude::*;

/// 这是一个 [`Message`]，任何观察它的观察者都会在它被激活时运行
#[derive(Message)]
struct MyMessage;

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_message::<MyMessage>()
        .add_systems(Startup, setup)
        .add_systems(Update, send_message)
        .add_observer(receive_message)
        .run();
}

fn send_message(mut commands: Commands) {
    // 发送消息
    commands.write_message(MyMessage);
}

fn receive_message(_message: On<MyMessage>) {
    info!("Message received!");
        }
```

**关键要点**：
- 使用 `Message` 派生宏定义消息
- 使用 `add_message()` 注册消息
- 使用 `write_message()` 发送消息
- 使用观察者接收消息

**注意事项**：
- 消息系统用于实现系统间通信
- 消息可以用于事件驱动逻辑
- 注意消息系统的性能影响

**最佳实践**：
- 对于系统间通信，使用消息系统
- 对于事件驱动逻辑，使用消息系统
- 注意消息系统的性能影响

### 错误处理

系统可以返回 `Result` 来处理错误。

**源代码文件**：`bevy/examples/ecs/error_handling.rs`

**代码示例**：

```rust
use bevy::ecs::error::warn;

fn main() {
    let mut app = App::new();
    // 默认情况下，返回错误的可失败系统会 panic
    //
    // 我们可以通过设置自定义错误处理器来改变这一点
    // 它适用于整个应用
    // 这里我们使用内置错误处理器之一
    app.set_error_handler(warn);

    app.add_plugins(DefaultPlugins);
    app.add_systems(Startup, setup);

    // 单个系统也可以通过管道输出结果来处理：
    app.add_systems(
        PostStartup,
        failing_system.pipe(|result: In<Result>| {
            let _ = result.0.inspect_err(|err| info!("captured error: {err}"));
        }),
    );

    app.run();
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

### 动态 ECS

动态 ECS 允许动态创建组件和实体。

**源代码文件**：`bevy/examples/ecs/dynamic.rs`

**代码示例**：

```rust
use bevy::prelude::*;

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Startup, setup)
        .add_systems(Update, query_dynamic_components)
        .run();
}

fn setup(mut commands: Commands, mut world: &mut World) {
    // 动态创建组件
    let component_id = world.register_component_with_descriptor(ComponentDescriptor::new(
        "MyDynamicComponent",
        StorageType::Table,
    ));

    // 使用动态组件创建实体
    let entity = commands.spawn_empty().id();
    commands.entity(entity).insert_by_id(component_id, MyData(42));
}

fn query_dynamic_components(world: &mut World, component_id: ComponentId) {
    // 查询动态组件
    let query = Query::new((component_id,));
    for (data,) in query.iter(world) {
        // ...
    }
}
```

**关键要点**：
- 使用 `register_component_with_descriptor()` 动态创建组件
- 使用 `insert_by_id()` 插入动态组件
- 使用 `Query` 查询动态组件
- 动态 ECS 可以用于运行时创建组件

**注意事项**：
- 动态 ECS 允许运行时创建组件
- 动态 ECS 可以用于插件系统
- 注意动态 ECS 的性能影响

**最佳实践**：
- 对于需要运行时创建组件的场景，使用动态 ECS
- 对于插件系统，使用动态 ECS
- 注意动态 ECS 的性能影响

## 实际应用

### 在游戏开发中的应用场景

ECS 进阶功能在游戏开发中有广泛的应用：

1. **性能优化**：使用并行查询提高系统执行效率
2. **变更响应**：使用变更检测和观察者响应组件变化
3. **关系管理**：使用关系系统建立实体间的复杂关系
4. **生命周期管理**：使用组件钩子管理组件的生命周期
5. **事件驱动**：使用消息系统和观察者实现事件驱动逻辑

### 常见问题

**问题 1**：何时使用变更检测，何时使用观察者？

**解决方案**：
- 变更检测适用于需要查询已变更组件的场景
- 观察者适用于需要响应组件生命周期事件的场景
- 对于简单场景，优先使用变更检测

**问题 2**：并行查询何时更快？

**解决方案**：
- 并行查询适用于计算密集型操作
- 对于简单操作，并行查询可能不会更快
- 需要根据实际情况测试性能

**问题 3**：如何优化查询组合的性能？

**解决方案**：
- 使用空间分区减少需要检查的实体对
- 使用查询过滤器减少查询的实体数量
- 考虑使用并行查询组合

### 性能考虑

1. **并行查询**：只在计算密集型操作时使用
2. **查询组合**：注意性能影响，考虑使用空间分区
3. **观察者**：考虑性能开销，优先使用变更检测
4. **组件钩子**：比观察者开销更小，但灵活性较低

## 相关资源

**相关源代码文件**：
- `bevy/examples/ecs/change_detection.rs` - 变更检测示例
- `bevy/examples/ecs/component_hooks.rs` - 组件生命周期钩子示例
- `bevy/examples/ecs/relationships.rs` - 关系系统示例
- `bevy/examples/ecs/parallel_query.rs` - 并行查询示例
- `bevy/examples/ecs/iter_combinations.rs` - 查询组合示例
- `bevy/examples/ecs/observers.rs` - 观察者模式示例
- `bevy/examples/ecs/removal_detection.rs` - 移除检测示例
- `bevy/examples/ecs/message.rs` - 消息系统示例
- `bevy/examples/ecs/error_handling.rs` - 错误处理示例
- `bevy/examples/ecs/dynamic.rs` - 动态 ECS 示例

**官方文档链接**：
- [Bevy ECS 官方文档](https://docs.rs/bevy_ecs/latest/bevy_ecs/)
- [变更检测文档](https://docs.rs/bevy_ecs/latest/bevy_ecs/change_detection/index.html)
- [关系系统文档](https://docs.rs/bevy_ecs/latest/bevy_ecs/relationship/index.html)
- [观察者文档](https://docs.rs/bevy_ecs/latest/bevy_ecs/observer/index.html)

**进一步学习建议**：
- 学习 Bevy 的其他高级功能，如自定义系统参数、系统调度等
- 阅读 Bevy ECS 源码，深入理解实现原理
- 实践编写自己的高级 ECS 系统，加深理解

---

**索引**：[返回上级目录](/wiki/BevyBook/ECS/)

