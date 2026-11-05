# 跨类关系索引：授予与激活

本页按所属类串联关系，成员和函数实现归入各类页面。核对日期：2026-11-05。

## 组件与 ActorInfo 所属关系

| 类 | 本类负责的内容 | 详细说明 |
| --- | --- | --- |
| AGAS_PlayerState | 创建自定义 ASC 与属性集，ASC 复制模式 Mixed | [PlayerState](../Player/PlayerState.md) |
| AGAS_PlayerCharacter | 两个 getter 转发、ActorInfo 初始化与本端广播，服务端授予及初值应用 | [玩家角色](../Characters/PlayerCharacter.md) |
| AGAS_EnemyCharacter | 自身创建 ASC / AS，BeginPlay 初始化及广播，服务端授予并应用初始 GE | [敌人角色](../Characters/EnemyCharacter.md) |
| AGAS_BaseCharacters | 委托声明、StartupAbilities、GiveAbility、InitializeAttributes | [角色基类](../Characters/CharacterBase.md) |

```mermaid
flowchart TD
    state["PlayerState 构造函数"] -->|"创建并持有，Mixed"| playerAsc["玩家 ASC"]
    possessed["服务端 PlayerCharacter.PossessedBy"] -->|"Owner 为 PlayerState，Avatar 为当前角色"| playerInfo["InitAbilityActorInfo"]
    playerAsc -->|"作为初始化目标"| playerInfo
    playerInfo -->|"先广播本端 ASC / AS"| playerSignal["OnASCInitialized.Broadcast"]
    playerSignal -->|"服务端广播后"| startup["GiveStartupAbilities"]
    onRep["客户端 PlayerCharacter.OnRep_PlayerState"] -->|"本端引用就绪且 ASC 有效时"| clientInfo["InitAbilityActorInfo：PlayerState / 当前角色"]
    clientInfo -->|"广播后结束客户端角色回调"| clientSignal["OnASCInitialized.Broadcast"]
    enemy["EnemyCharacter 构造函数"] -->|"创建并持有，Minimal"| enemyAsc["敌人 ASC"]
    begin["EnemyCharacter.BeginPlay：各端"] -->|"本端 ASC 有效，Owner 与 Avatar 均为 this"| enemyInfo["InitAbilityActorInfo"]
    enemyAsc -->|"作为初始化目标"| enemyInfo
    enemyInfo -->|"各端先广播本端 ASC / AS"| enemySignal["OnASCInitialized.Broadcast"]
    enemySignal -->|"仅 HasAuthority 为真时继续"| startup
    startup -->|"取得各自 ASC，遍历能力类构造 Spec"| give["ASC.GiveAbility"]
    startup -->|"服务端授予循环结束后"| initAttributes["InitializeAttributes"]
    initAttributes -->|"创建等级 1 的 Spec 并应用到自身"| values["初始 GE 改变属性数据"]
```

客户端的角色初始化回调不授予能力或调用 `InitializeAttributes()`；它和 ASC 的能力列表复制回调是不同事件，不能视为固定串行的两个步骤。

## 自动激活与输入激活的类边界

| 路径 | 入口所属类 | 传递信息 | 后续调用 |
| --- | --- | --- | --- |
| 授予回调 | UGAS_AbilitySystemComponent.OnGiveAbility | 本次 Spec | HandleAutoActivateAbility |
| 能力列表复制 | UGAS_AbilitySystemComponent.OnRep_ActivateAbilities | 锁内逐项 Spec | HandleAutoActivateAbility |
| 玩家输入 | AGAS_PlayerController.Primary / Secondary / Tertiary | 能力检索 Tag | ActivateAbility / TryActivateAbilitiesByTag |

```mermaid
flowchart TD
    give["ASC.GiveAbility"] -->|"引擎授予回调"| onGive["自定义 ASC.OnGiveAbility"]
    onGive -->|"本次 Spec，先执行父类后"| handler["HandleAutoActivateAbility"]
    rep["客户端 ASC.OnRep_ActivateAbilities"] -->|"父类执行后，列表锁内逐项 Spec"| handler
    handler -->|"精确匹配 ActivateOnGiven，按 Handle"| direct["TryActivateAbility"]
    input["本地输入 Started"] -->|"调用 Controller 对应能力回调"| tag["ActivateAbility：能力 Tag"]
    tag -->|"通过 Pawn 接口取得 PlayerState.ASC"| query["TryActivateAbilitiesByTag"]
    query -->|"检索已授予能力，尝试候选"| direct
    direct -->|"上下文、网络策略和激活条件允许时"| ability["能力激活"]
```

这里存在自动激活的实际 C++ 处理代码，但尝试调用不保证成功。组件没有“一次性激活”记录或失败队列；具体实现见 [自定义 ASC](AbilitySystemComponent.md)。输入路径见 [PlayerController](../Player/PlayerController.md)。

## 属性复制与 Widget 引用缓存

| 类 / 函数 | 实际完成的事情 | 所属说明 |
| --- | --- | --- |
| BaseCharacters.InitializeAttributes | 在服务端应用配置的初始 GE | [角色基类](../Characters/CharacterBase.md) |
| UGAS_AttributeSet | Health / MaxHealth / Mana / MaxMana 访问器和复制通知 | [属性集](AttributeSet.md) |
| UGAS_WidgetComponent.InitAbilitySystemData | 缓存角色、ASC、属性集弱引用 | [Widget 组件](../UI/WidgetComponent.md) |
| WidgetComponent.OnSACInitialized | 根据委托参数更新引用 | 同上 |

```mermaid
flowchart TD
    widget["Widget.BeginPlay"] -->|"先尝试取得角色 ASC / AS"| pointers{"指针缓存都有效？"}
    pointers -->|"否，Owner 须为有效角色"| bind["绑定 OnASCInitialized"]
    role["角色本端 ActorInfo 初始化"] -->|"先广播"| signal["OnASCInitialized：ASC / AS"]
    bind -->|"收到本端广播时调用"| callback["Widget.OnSACInitialized"]
    signal -->|"把指针传给已绑定回调"| callback
    pointers -->|"是，不绑定通知"| cached["引用可用，本次 BeginPlay 结束"]
    signal -->|"服务端随后授予能力，再应用"| initial["InitializeAttributes"]
    initial -->|"效果配置决定数值"| attributes["UGAS_AttributeSet"]
    attributes -->|"满足网络复制条件时"| onRep["客户端属性 OnRep，协调所属 ASC"]
```

创建对象、初始化 ActorInfo、应用初始属性值是不同阶段。角色委托和 Widget 的指针检查均不能表示初始 GE 已完成；Widget 源码明确留下这部分 TODO。当前顺序还意味着授予回调中的自动激活尝试可能先于本次属性初值应用。

## 能力类之间的关系

[UGAS_GameplayAbility](GameplayAbility.md) 在激活回调中按 `bDrawDebugs` 显示提示；[UGAS_Primary](Primary.md) 查询并返回去重 Actor，另由 `SendHitReactEventToActors()` 发送事件。[UGAS_HitReact](HitReact.md) 缓存 AvatarForward 与 ToInstigator，[方向工具类](../Utils/BlueprintLibrary.md) 判定枚举并映射名称。各函数的蓝图连接及接收端配置见 [事件关系索引](GameplayEvents.md)。

## 排查位置索引

| 检查对象 | 所属类或文件 |
| --- | --- |
| 实际玩家对象类型 | GameMode 蓝图默认类与地图覆盖 |
| 输入资源引用、事件绑定 | PlayerController.h / SetupInputComponent |
| ASC getter 返回值、Owner / Avatar | PlayerState、PlayerCharacter、EnemyCharacter |
| 启动能力类数组 | BaseCharacters.h 与角色蓝图默认值 |
| 自动激活标签匹配与尝试结果 | AbilitySystemComponent.cpp |
| 查询条件、球心与碰撞参数 | Primary.cpp |
| 事件发送 | Primary.SendHitReactEventToActors，目标数组及 Payload.Instigator |
| 事件接收与缓存调用 | 目标 ASC 与能力蓝图；HitReact.CacheHitDirectionVector 调用时机待核验 |
| 前后左右与名称 | BlueprintLibrary.GetHitDirection / GetHitDirectionName |

初始化属性效果未配置时，检查 BaseCharacters 的 `InitializeAttributesEffect`；界面没有数值变化时，先确认 Widget 目前只缓存引用，属性监听与刷新并未在该类 C++ 中实现。
