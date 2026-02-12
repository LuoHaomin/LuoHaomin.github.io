# 查询（Queries）## 概述

**学习目标**：
- 理解查询的概念和作用
- 掌握如何定义和使用查询
- 了解查询过滤器的使用方法
- 理解自定义查询参数和查询组合

**前置知识要求**：
- 核心编程框架（ECS）
- 组件（Components）
- 实体（Entities）
- 系统（Systems）
- Rust 基础语法


## 核心概念

### 什么是查询？

查询用于访问具有特定组件的实体。查询会自动过滤出具有指定组件的实体，并允许系统对这些实体进行操作。

**为什么使用查询？**

1. **自动过滤**：查询自动过滤出具有指定组件的实体
2. **高效访问**：查询提供高效的组件访问方式
3. **并行执行**：查询可以并行执行，提高性能

### 查询的设计思想

查询采用数据导向的设计思想，将数据（组件）和逻辑（系统）分离。这种设计使得：

- 系统可以高效访问组件
- 查询可以并行执行，提高性能
- 查询可以灵活组合，实现复杂的查询需求

## 基础用法

### 基本查询

使用 `Query<T>` 查询具有特定组件的实体。

**源代码文件**：`bevy/examples/ecs/ecs_guide.rs`

**代码示例**：

```rust
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
- 使用 `Query<T>` 查询具有特定组件的实体
- `&T` 表示不可变访问组件
- `&mut T` 表示可变访问组件
- 查询可以同时访问多个组件
- 使用 `Mut<T>` 类型进行可变访问

**说明**：
查询是访问组件的主要方式。查询会自动过滤出具有指定组件的实体，并允许系统对这些实体进行操作。查询可以并行执行，提高性能。

### 查询过滤器

使用查询过滤器进一步过滤查询结果。

**源代码文件**：`bevy/examples/ecs/change_detection.rs`

**代码示例**：

```rust
/// 查询过滤器如 [`Changed<T>`] 和 [`Added<T>`] 确保只有匹配这些过滤器的实体
/// 才会被查询返回
///
/// 使用 [`Ref<T>`] 系统参数允许你访问变更检测信息，但不会过滤查询
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
}
```

**关键要点**：
- 使用 `Changed<T>` 过滤器查询已变更的组件
- 使用 `Added<T>` 过滤器查询新添加的组件
- 使用 `With<T>` 过滤器查询具有特定组件的实体
- 使用 `Without<T>` 过滤器查询不具有特定组件的实体
- 使用 `Ref<T>` 访问变更检测信息，但不过滤查询

**说明**：
查询过滤器用于进一步过滤查询结果。使用 `Changed<T>` 过滤器可以只查询已变更的组件，这对于性能优化很重要。使用 `Added<T>` 过滤器可以只查询新添加的组件。

## 进阶用法

### 自定义查询参数

使用 `QueryData` 派生宏定义自定义查询类型，避免使用元组。

**源代码文件**：`bevy/examples/ecs/custom_query_param.rs`

**代码示例**：

```rust
use bevy::{
    ecs::query::{QueryData, QueryFilter},
    prelude::*,
};

#[derive(Component, Debug)]
struct ComponentA;
#[derive(Component, Debug)]
struct ComponentB;
#[derive(Component, Debug)]
struct ComponentC;
#[derive(Component, Debug)]
struct ComponentD;

/// 虽然常规元组查询在大多数简单场景中工作得很好
/// 但使用声明为命名结构体的自定义查询可以带来以下优势：
/// - 它们有助于避免解构或使用 `q.0, q.1, ...` 访问模式
/// - 使用结构体添加、移除组件或更改项目顺序大大减少了维护负担
/// - 命名结构体支持组合模式，使查询类型更容易重用
/// - 你可以绕过查询元组存在的 15 个组件的限制
#[derive(QueryData)]
#[query_data(derive(Debug))]
struct ReadOnlyCustomQuery<T: Component + Debug, P: Component + Debug> {
    entity: Entity,
    a: &'static ComponentA,
    b: Option<&'static ComponentB>,
    nested: NestedQuery,
    optional_nested: Option<NestedQuery>,
    optional_tuple: Option<(&'static ComponentB, &'static ComponentZ)>,
    generic: GenericQuery<T, P>,
    empty: EmptyQuery,
}

#[derive(QueryData)]
#[query_data(derive(Debug))]
struct NestedQuery {
    c: &'static ComponentC,
    d: Option<&'static ComponentD>,
}

#[derive(QueryData)]
#[query_data(derive(Debug))]
struct GenericQuery<T: Component, P: Component> {
    generic: (&'static T, &'static P),
}

#[derive(QueryFilter)]
struct CustomQueryFilter<T: Component, P: Component> {
    _c: With<ComponentC>,
    _d: With<ComponentD>,
    _or: Or<(Added<ComponentC>, Changed<ComponentD>, Without<ComponentZ>)>,
    _generic_tuple: (With<T>, With<P>),
}

fn print_components_read_only(
    query: Query<
        ReadOnlyCustomQuery<ComponentC, ComponentD>,
        CustomQueryFilter<ComponentC, ComponentD>,
    >,
) {
    println!("Print components (read_only):");
    for e in &query {
        println!("Entity: {}", e.entity);
        println!("A: {:?}", e.a);
        println!("B: {:?}", e.b);
        println!("Nested: {:?}", e.nested);
        println!("Optional nested: {:?}", e.optional_nested);
        println!("Optional tuple: {:?}", e.optional_tuple);
        println!("Generic: {:?}", e.generic);
    }
    println!();
}
```

**关键要点**：
- 使用 `QueryData` 派生宏定义自定义查询类型
- 自定义查询类型可以避免使用元组
- 自定义查询类型可以支持组合模式
- 自定义查询类型可以绕过 15 个组件的限制
- 使用 `QueryFilter` 派生宏定义自定义过滤器类型

**注意事项**：
- 自定义查询类型需要实现 `QueryData` trait
- 自定义过滤器类型需要实现 `QueryFilter` trait
- 使用 `#[query_data(mutable)]` 属性定义可变查询

**最佳实践**：
- 对于复杂的查询，使用自定义查询类型
- 对于需要重用的查询，使用自定义查询类型
- 对于超过 15 个组件的查询，使用自定义查询类型

### 查询组合

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

### 并行查询

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
    let left = width / -2.0;
    let right = width / 2.0;
    let bottom = height / -2.0;
    let top = height / 2.0;
    // 也可以覆盖默认批次大小
    // 在这种情况下，选择批次大小为 32 以限制 ParallelIterator 的开销
    // 因为取反向量非常便宜
    sprites
        .par_iter_mut()
        .batching_strategy(BatchingStrategy::fixed(32))
        .for_each(|(transform, mut v)| {
            if !(left < transform.translation.x
                && transform.translation.x < right
                && bottom < transform.translation.y
                && transform.translation.y < top)
            {
                // 为简单起见，只反转速度；不使用真实的反弹
                v.0 = -v.0;
            }
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

## 实际应用

### 在游戏开发中的应用场景

查询在游戏开发中有广泛的应用：

1. **游戏对象访问**：访问玩家、敌人、道具等游戏对象
2. **碰撞检测**：使用查询组合检测实体间的碰撞
3. **性能优化**：使用并行查询提高系统执行效率

### 常见问题

**问题 1**：如何查询具有多个组件的实体？

**解决方案**：使用元组查询多个组件，如 `Query<(&ComponentA, &ComponentB)>`。

**问题 2**：如何查询不具有特定组件的实体？

**解决方案**：使用 `Without<T>` 过滤器，如 `Query<&ComponentA, Without<ComponentB>>`。

**问题 3**：如何优化查询性能？

**解决方案**：
- 只查询需要的组件
- 使用查询过滤器减少查询的实体数量
- 对于计算密集型操作，使用并行查询

### 性能考虑

1. **查询优化**：只查询需要的组件，减少查询开销
2. **并行执行**：查询可以并行执行，但可变访问会阻止并行
3. **查询组合**：注意性能影响，考虑使用空间分区

## 相关资源

**相关源代码文件**：
- `bevy/examples/ecs/ecs_guide.rs` - ECS 完整指南示例（查询使用）
- `bevy/examples/ecs/change_detection.rs` - 变更检测示例（查询过滤器）
- `bevy/examples/ecs/custom_query_param.rs` - 自定义查询参数示例
- `bevy/examples/ecs/iter_combinations.rs` - 查询组合示例
- `bevy/examples/ecs/parallel_query.rs` - 并行查询示例

**官方文档链接**：
- [Bevy Query 官方文档](https://docs.rs/bevy_ecs/latest/bevy_ecs/query/index.html)
- [QueryData 文档](https://docs.rs/bevy_ecs/latest/bevy_ecs/query/trait.QueryData.html)
- [QueryFilter 文档](https://docs.rs/bevy_ecs/latest/bevy_ecs/query/trait.QueryFilter.html)

**进一步学习建议**：
- 学习系统（Systems），了解如何在系统中使用查询
- 学习组件（Components），了解如何定义组件
- 学习 ECS 进阶，了解查询的高级功能

---

**索引**：[返回上级目录](/wiki/bevybook/ecs/)


