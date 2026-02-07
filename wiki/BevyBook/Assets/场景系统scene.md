# 场景系统（Scene）## 概述

**学习目标**：
- 理解 Bevy 场景系统的基本概念
- 掌握场景的保存和加载
- 了解动态场景的使用
- 学会使用场景序列化和反序列化

**前置知识要求**：
- Bevy 快速入门
- ECS 基础
- 资源管理基础
- 反射系统基础


## 核心概念

### 什么是场景系统？

场景系统是 Bevy 中用于保存和加载场景数据的功能。场景系统允许您将实体、组件和资源序列化到文件中，并在需要时加载它们。

**为什么需要场景系统？**

1. **场景保存**：场景系统可以保存游戏场景到文件
2. **场景加载**：场景系统可以从文件加载游戏场景
3. **数据持久化**：场景系统可以实现数据持久化
4. **场景编辑**：场景系统可以用于场景编辑工具

### 场景系统的核心组件

Bevy 场景系统包含以下核心组件：

- **Scene**：场景，包含序列化的实体和组件
- **DynamicScene**：动态场景，用于运行时创建场景
- **DynamicSceneRoot**：动态场景根，用于加载场景
- **SceneSpawner**：场景生成器，用于生成场景实体

## 基础用法

### 场景保存

保存场景到文件。

**源代码文件**：`bevy/examples/scene/scene.rs`

**代码示例**：

```rust
use bevy::{asset::LoadState, prelude::*, tasks::IoTaskPool};
use core::time::Duration;
use std::{fs::File, io::Write};

/// 入口点
///
/// 设置默认插件，注册所有必要的组件/资源类型
/// 以进行序列化/反射，并在正确的调度中运行各种系统。
fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(
            Startup,
            (save_scene_system, load_scene_system, infotext_system),
        )
        .add_systems(Update, (log_system, panic_on_fail))
        .run();
}

/// 演示如何从头开始创建新场景，填充数据，
/// 然后将其序列化到文件。新文件写入 `NEW_SCENE_FILE_PATH`。
///
/// 此系统创建一个新世界，复制类型注册表以便我们的
/// 自定义组件类型被识别，生成一些示例实体和资源，
/// 然后序列化生成的动态场景。
fn save_scene_system(world: &mut World) {
    // 场景可以从任何 ECS World 创建。
    // 您可以创建一个新场景或使用当前 World。
    // 为了演示目的，我们将创建一个新的。
    let mut scene_world = World::new();

    // `TypeRegistry` 资源包含有关所有已注册类型（包括组件）的信息。
    // 这用于构建场景，因此我们希望确保之前的类型注册
    // 也存在于这个新场景世界中。
    // 为此，我们可以简单地克隆 `AppTypeRegistry` 资源。
    let type_registry = world.resource::<AppTypeRegistry>().clone();
    scene_world.insert_resource(type_registry);

    let mut component_b = ComponentB::from_world(world);
    component_b.value = "hello".to_string();
    scene_world.spawn((
        component_b,
        ComponentA { x: 1.0, y: 2.0 },
        Transform::IDENTITY,
        Name::new("joe"),
    ));
    scene_world.spawn(ComponentA { x: 3.0, y: 4.0 });
    scene_world.insert_resource(ResourceA { score: 1 });

    // 准备好我们的示例世界后，我们现在可以使用 DynamicScene 或 DynamicSceneBuilder 创建场景。
    // 为了简单起见，我们将使用 DynamicScene 创建场景：
    let scene = DynamicScene::from_world(&scene_world);

    // 场景可以像这样序列化：
    let type_registry = world.resource::<AppTypeRegistry>();
    let type_registry = type_registry.read();
    let serialized_scene = scene.serialize(&type_registry).unwrap();

    // 在控制台中显示场景
    info!("{}", serialized_scene);

    // 将场景写入新文件。使用任务避免在系统中调用文件系统 API，
    // 因为它们是阻塞的。
    //
    // 这在 Wasm 中无法工作，因为没有文件系统访问。
    #[cfg(not(target_arch = "wasm32"))]
    IoTaskPool::get()
        .spawn(async move {
            // 写入场景文件
            File::create("scenes/load_scene_example-new.scn.ron")
                .and_then(|mut file| file.write_all(serialized_scene.as_bytes()))
                .expect("Error while writing scene to file");
        })
        .detach();
}

/// 从资源文件加载场景并将其动态应用到 Bevy World 中的实体。
///
/// 生成 `DynamicSceneRoot` 会创建一个新的父实体，然后生成场景的
/// 新实例作为其子实体。如果您修改 `SCENE_FILE_PATH` 场景文件，
/// 或者如果您启用文件监视，您可以看到更改立即反映。
fn load_scene_system(mut commands: Commands, asset_server: Res<AssetServer>) {
    commands.spawn(DynamicSceneRoot(asset_server.load("scenes/load_scene_example.scn.ron")));
}

/// 记录对 `ComponentA` 实体所做的更改，并检查 `ResourceA` 是否最近被添加。
///
/// 每当修改 `ComponentA` 时，该更改将出现在这里。此系统
/// 演示了如何在运行时检测和处理场景更新。
fn log_system(
    query: Query<(Entity, &ComponentA), Changed<ComponentA>>,
    res: Option<Res<ResourceA>>,
) {
    for (entity, component_a) in &query {
        info!("  Entity({})", entity.index());
        info!(
            "    ComponentA: {{ x: {} y: {} }}\n",
            component_a.x, component_a.y
        );
    }
    if let Some(res) = res
        && res.is_added()
    {
        info!("  New ResourceA: {{ score: {} }}\n", res.score);
    }
}

/// 一个示例组件，完全可序列化。
///
/// 此组件具有将包含在场景文件中的公共 `x` 和 `y` 字段。
/// 注意它如何派生 `Default`、`Reflect`，并使用 `#[reflect(Component)]` 声明自己为反射组件。
#[derive(Component, Reflect, Default)]
#[reflect(Component)] // 这告诉反射派生也反射组件行为
struct ComponentA {
    /// 示例 `f32` 字段
    pub x: f32,
    /// 另一个示例 `f32` 字段
    pub y: f32,
}

/// 一个示例组件，包括可序列化和不可序列化字段。
///
/// 这对于跳过运行时数据的序列化或您不想要写入场景文件的字段很有用。
#[derive(Component, Reflect)]
#[reflect(Component)]
struct ComponentB {
    /// 将被序列化的字符串字段。
    pub value: String,
    /// 一个 `Duration` 字段，永远不应该序列化到场景文件，所以我们跳过它。
    #[reflect(skip_serializing)]
    pub _time_since_startup: Duration,
}

/// 这为 `ComponentB` 实现了 `FromWorld`，让我们通过访问当前 ECS 资源来初始化运行时字段。
/// 在这种情况下，我们获取 `Time` 资源并存储当前经过的时间。
impl FromWorld for ComponentB {
    fn from_world(world: &mut World) -> Self {
        let time = world.resource::<Time>();
        ComponentB {
            _time_since_startup: time.elapsed(),
            value: "Default Value".to_string(),
        }
    }
}

/// 一个简单的资源，也派生 `Reflect`，允许它存储在场景中。
///
/// 就像组件一样，如果需要，您可以跳过序列化字段或实现 `FromWorld`。
#[derive(Resource, Reflect, Default)]
#[reflect(Resource)]
struct ResourceA {
    /// 此资源跟踪 `score` 值。
    pub score: u32,
}
```

**关键要点**：
- 使用 `DynamicScene::from_world()` 从世界创建场景
- 使用 `scene.serialize()` 序列化场景
- 使用 `DynamicSceneRoot` 加载场景
- 使用 `#[reflect(Component)]` 和 `#[reflect(Resource)]` 使类型可序列化
- 使用 `#[reflect(skip_serializing)]` 跳过序列化字段

**说明**：
场景保存是场景系统的基础。通过使用场景保存，可以将游戏场景保存到文件，实现数据持久化。

### 场景加载

从文件加载场景。

**关键信息**：
- 使用 `AssetServer::load()` 加载场景文件
- 使用 `DynamicSceneRoot` 生成场景根实体
- 场景会自动生成为场景根的子实体
- 使用文件监视可以实时更新场景

**说明**：
场景加载是场景系统的基础。通过使用场景加载，可以从文件加载游戏场景，实现场景的复用。

## 进阶用法

### 动态场景

使用动态场景在运行时创建场景。

**关键信息**：
- 使用 `DynamicScene::from_world()` 从世界创建动态场景
- 使用 `DynamicSceneBuilder` 构建动态场景
- 使用 `DynamicScene::write_to_world()` 将场景写入世界

**说明**：
动态场景是场景系统的高级功能。通过使用动态场景，可以在运行时创建场景，实现更灵活的场景管理。

### 场景序列化

使用反射系统序列化和反序列化场景。

**关键信息**：
- 使用 `#[derive(Reflect)]` 使类型可反射
- 使用 `#[reflect(Component)]` 使组件可序列化
- 使用 `#[reflect(Resource)]` 使资源可序列化
- 使用 `#[reflect(skip_serializing)]` 跳过序列化字段

**说明**：
场景序列化是场景系统的重要功能。通过使用反射系统，可以序列化和反序列化场景，实现场景的保存和加载。

## 实际应用

### 在游戏开发中的应用场景

场景系统在游戏开发中有广泛的应用：

1. **场景保存**：保存游戏场景到文件
2. **场景加载**：从文件加载游戏场景
3. **数据持久化**：实现游戏数据的持久化
4. **场景编辑**：用于场景编辑工具

### 常见问题

**问题 1**：如何使类型可序列化？

**解决方案**：
- 使用 `#[derive(Reflect)]` 使类型可反射
- 使用 `#[reflect(Component)]` 使组件可序列化
- 使用 `#[reflect(Resource)]` 使资源可序列化
- 使用 `AppTypeRegistry` 注册类型

**问题 2**：如何跳过序列化字段？

**解决方案**：
- 使用 `#[reflect(skip_serializing)]` 跳过序列化字段
- 使用 `FromWorld` 实现初始化运行时字段

**问题 3**：如何加载场景？

**解决方案**：
- 使用 `AssetServer::load()` 加载场景文件
- 使用 `DynamicSceneRoot` 生成场景根实体
- 场景会自动生成为场景根的子实体

### 性能考虑

1. **场景序列化**：场景序列化是阻塞操作，应使用异步任务
2. **场景加载**：场景加载是异步操作，应使用 `AssetServer`
3. **类型注册**：类型注册应在应用启动时完成

## 相关资源

**相关源代码文件**：
- `bevy/examples/scene/scene.rs` - 场景系统示例

**官方文档链接**：
- [Bevy 场景系统](https://docs.rs/bevy/latest/bevy/scene/index.html)
- [Bevy 反射系统](https://docs.rs/bevy/latest/bevy/reflect/index.html)

**进一步学习建议**：
- 学习资源管理，了解资源加载和管理
- 学习反射系统，了解类型反射和序列化

---

**索引**：[返回上级目录](/wiki/bevybook/assets/)

```

```markdown
## 内容列表

### 1. [资源管理](/wiki/bevybook/assets/资源管理)

- 资源系统概述
- 资源类型
- 资源加载
- 资源管理

**学习目标**：能够加载和管理资源


### 2. [场景系统（Scene）](/wiki/bevybook/assets/场景系统（Scene）)

- 场景保存
- 场景加载
- 动态场景
- 场景序列化

**学习目标**：能够保存和加载场景


