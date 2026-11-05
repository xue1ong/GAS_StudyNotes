# GAS_StudyNotes

按参考项目 `..\GAS_Learning` 当前类文件组织的 GAS 学习笔记。核对日期：2026-11-05；项目声明引擎版本：**Unreal Engine 5.6**。

## 类文件索引

下表路径以 `Source/GAS_Learning` 为根；每个类均有同名 `.h` / `.cpp`。各类页面分别说明头文件中的继承、成员和函数，以及源文件的实现。

| 类文件 | 类 | 笔记 |
| --- | --- | --- |
| `Characters/GAS_BaseCharacters` | `AGAS_BaseCharacters` | [数据接口、委托、启动能力与初始属性效果](Characters/CharacterBase.md) |
| `Characters/GAS_PlayerCharacter` | `AGAS_PlayerCharacter` | [摄像机、ASC / AS 访问与两端初始化](Characters/PlayerCharacter.md) |
| `Characters/Enemy/GAS_EnemyCharacter` | `AGAS_EnemyCharacter` | [自身 ASC、BeginPlay 与服务端授予](Characters/EnemyCharacter.md) |
| `Player/GAS_PlayerState` | `AGAS_PlayerState` | [玩家 ASC / AS 的创建与复制设置](Player/PlayerState.md) |
| `Player/GAS_PlayerController` | `AGAS_PlayerController` | [输入成员、绑定与能力激活](Player/PlayerController.md) |
| `AbilitySystem/GAS_AttributeSet` | `UGAS_AttributeSet` | [属性、访问器宏与复制通知](AbilitySystem/AttributeSet.md) |
| `UI/GAS_WidgetComponent` | `UGAS_WidgetComponent` | [弱引用缓存与角色初始化通知](UI/WidgetComponent.md) |
| `AbilitySystem/GAS_AbilitySystemComponent` | `UGAS_AbilitySystemComponent` | [授予回调、复制回调与自动激活检查](AbilitySystem/AbilitySystemComponent.md) |
| `AbilitySystem/Abilities/GAS_GameplayAbility` | `UGAS_GameplayAbility` | [激活回调与公共调试属性](AbilitySystem/GameplayAbility.md) |
| `AbilitySystem/Abilities/Player/GAS_Primary` | `UGAS_Primary` | [命中查询、事件发送与调试绘制](AbilitySystem/Primary.md) |
| `AbilitySystem/Abilities/Enemy/GAS_HitReact` | `UGAS_HitReact` | [方向向量缓存与调用边界](AbilitySystem/HitReact.md) |
| `Utils/GAS_BlueprintLibrary` | `UGAS_BlueprintLibrary` | [方向枚举、判定与名称映射](Utils/BlueprintLibrary.md) |

## 非类文件与关系索引

| 文件 / 范围 | 笔记 |
| --- | --- |
| `GameplayTags/GAS_Tags.h/.cpp` 命名空间 | [原生标签声明、定义与消费位置](Tags/GameplayTags.md) |
| `.uproject`、Build / Target、`Config` 和业务资源 | [配置文件与资源索引](Game/ProjectSetup.md) |
| `BP_GAS_Gamemode` 蓝图及默认配置 | [GameMode 配置关系](Game/GameMode.md) |
| 角色、PlayerState、ASC 与 Controller | [跨类授予及激活关系](AbilitySystem/StartupAbilities.md) |
| 标签、能力类、动画与通知资产 | [事件关系索引](AbilitySystem/GameplayEvents.md) |

## 类间职责关系

```mermaid
flowchart TD
    base["AGAS_BaseCharacters"] -->|"被继承，共用 StartupAbilities 和授予函数"| player["AGAS_PlayerCharacter"]
    base -->|"被继承，共用授予函数"| enemy["AGAS_EnemyCharacter"]
    state["AGAS_PlayerState"] -->|"创建并持有，Mixed"| playerAsc["玩家 UGAS_AbilitySystemComponent"]
    player -->|"getter 转发到 PlayerState"| playerAsc
    enemy -->|"创建并持有，Minimal"| enemyAsc["敌人 UGAS_AbilitySystemComponent"]
    player -->|"服务端 PossessedBy：ActorInfo 初始化后"| notify["OnASCInitialized：先广播 ASC / AS"]
    notify -->|"服务端广播之后"| grant["GiveStartupAbilities"]
    enemy -->|"BeginPlay：各端初始化后广播"| notify
    grant -->|"构造 Spec 后调用 GiveAbility"| callback["自定义 ASC.OnGiveAbility"]
    grant -->|"服务端授予循环结束后"| initial["InitializeAttributes：初始 GE 应用"]
    notify -->|"本端已绑定的 Widget 接收"| widget["UGAS_WidgetComponent：更新引用"]
    initial -->|"通过各自 ASC 更新"| attributes["UGAS_AttributeSet：四个属性"]
    callback -->|"精确匹配 ActivateOnGiven 时"| auto["TryActivateAbility"]
    replicated["客户端 ASC.OnRep_ActivateAbilities"] -->|"列表锁内逐项执行同一标签检查"| auto
    controller["AGAS_PlayerController"] -->|"当前 Pawn 接口返回玩家 ASC，传入输入 Tag"| byTag["TryActivateAbilitiesByTag"]
    playerAsc -->|"提供已授予的能力候选"| byTag
```

图中的回调分别对应本端授予和客户端能力列表复制，不表示服务端与客户端存在固定完成顺序。自动激活是尝试调用，仍受能力激活条件、上下文和网络设置约束。

## 内容依据

代码以参考项目的 `.h` / `.cpp` 为准，配置以 `.uproject` / `.ini` 为准。短代码注明源文件、函数与节选范围；跨类页面只索引关系，具体函数归入所属类说明。

资源路径已核对文件存在；本次没有通过 UE 编辑器核验蓝图父类、默认数组、事件图和动画通知。未核验的关系以虚线标注。当前业务能力资源包含 `GA_GAS_Primary`、`GA_Secondary`、`GA_Teriary`、`GA_GAS_HitReact` 及其 Melee / Ranged 资源，其中 `GA_Teriary` 是实际文件拼写。

源码涵盖自动激活、命中查询、受击事件与方向处理，以及四个属性的复制、初始 GE 应用和 Widget 数据引用缓存。初始效果资产为 `GE_Initialize_Attributes`，具体属性数值、角色引用和界面组件挂载待编辑器核验；Widget 当前没有属性变化绑定或界面数值刷新实现，源码也没有伤害结算。

`OnASCInitialized` 的广播早于本次初始 GE 应用；Widget 的 `IsASCInitialized()` 只判断指针有效。这两处均不能单独表示属性初值已就绪。查询与发送、事件监听与方向缓存、方向名称与动画选择之间的蓝图连接仍待核验。完整关系见 [事件与受击方向索引](AbilitySystem/GameplayEvents.md)。

项目模块与 Source 目录使用 `GAS_Learning`，业务 Content 和地图资源路径仍使用 `GAS_Acquaintance`；脚本包重定向见 [项目配置](Game/ProjectSetup.md)。
