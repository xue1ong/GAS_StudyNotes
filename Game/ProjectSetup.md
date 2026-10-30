# 项目配置与资源入口

本页记录本地 UE 项目已确认的接入条件，便于复习或重新搭建工程。

最近核对日期：2026-10-29。参考目录为 `..\GAS_Acquaintance`；资源清单已同步玩家蓝图目录调整以及新增敌人、动画和通知资产。

## 引擎、插件与模块

`GAS_Acquaintance.uproject` 声明 `EngineAssociation: "5.6"`，运行时模块名为 `GAS_Acquaintance`，启用了 `GameplayAbilities` 插件。

`Source/GAS_Acquaintance/GAS_Acquaintance.Build.cs` 的公开依赖包括：

```csharp
"Core", "CoreUObject", "Engine", "InputCore",
"EnhancedInput", "GameplayAbilities", "GameplayTags", "GameplayTasks"
```

| 依赖 | 当前源码中的用途 |
| --- | --- |
| `EnhancedInput` | 输入 Action、Mapping Context、输入组件和本地玩家子系统 |
| `GameplayAbilities` | 自定义 ASC 与能力基类、GAS 接口和能力 Spec |
| `GameplayTags` | 原生标签声明 / 定义、Tag 容器 |
| `GameplayTasks` | 已列入 GAS 相关模块依赖；当前自定义 C++ 没有编写 Task |

插件启用和 C++ 模块依赖是两个检查点。当前 Game / Editor 两个 Target 均使用 `BuildSettingsVersion.V5` 和 `EngineIncludeOrderVersion.Unreal5_6`。

```mermaid
flowchart TD
    project[".uproject"] -->|"声明引擎关联"| engine["UE 5.6"]
    project -->|"启用插件"| plugin["GameplayAbilities"]
    build["GAS_Acquaintance.Build.cs"] -->|"声明 GAS 编译依赖"| gas["GameplayAbilities / GameplayTags / GameplayTasks"]
    build -->|"声明输入编译依赖"| enhanced["EnhancedInput"]
    plugin -->|"提供 GAS 插件模块"| gas
    gas -->|"支持 ASC、能力 Spec 与原生 Tag 的使用"| gameplay["角色与 PlayerState 的 GAS 接入"]
    enhanced -->|"提供输入组件与本地子系统 API"| controller["Controller 输入绑定"]
    inputConfig["DefaultInput.ini"] -->|"选择 Enhanced Input 默认类"| controller
```

这里的边表示配置和编译依赖，不表示运行时初始化的先后顺序。

## 默认地图与输入类型

`Config/DefaultEngine.ini` 将启动地图和默认游戏地图设为 `Map_Main`，默认 GameMode 设为 `BP_GAS_Gamemode`，详情见 [GameMode](GameMode.md)。

`Config/DefaultInput.ini` 包含：

```ini
DefaultPlayerInputClass=/Script/EnhancedInput.EnhancedPlayerInput
DefaultInputComponentClass=/Script/EnhancedInput.EnhancedInputComponent
```

这与 Controller 将 `InputComponent` 转型为 `UEnhancedInputComponent` 的实现对应。具体输入键位在 IMC 资产中，本页不根据 Action 名推测按键。

## 业务资源目录

以下路径以 `Content/GAS_Acquaintance` 为根：

```text
Maps/Map_Main.umap
Blueprints/
  Game/BP_GAS_Gamemode.uasset
  Characters/
    Player/
      BP_GAS_PlayerCharacter.uasset
      Anim/AM_GAS_Primary_A.uasset
      Anim/AM_GAS_Primary_B.uasset
      Anim/Primary_A.uasset
      Anim/Primary_B.uasset
    Enemy/
      BP_Enemy_Melee.uasset
      BP_Enemy_Ranged.uasset
      Anim/ABP_Enemy_Melee.uasset
      Anim/ABP_Enemy_Ranged.uasset
  Player/BP_GAS_PlayerController.uasset
  Player/BP_GAS_PlayerState.uasset
  Notifies/AN_SendEventToActor.uasset
  Input/
    IMC_Movement.uasset
    IMC_Abilities.uasset
    MovementActions/IA_Jump.uasset
    MovementActions/IA_Move.uasset
    MovementActions/IA_Look.uasset
    AbilityActions/IA_Primary.uasset
    AbilityActions/IA_Secondary.uasset
    AbilityActions/IA_Tertiary.uasset
  AbilitySystem/PlayerAbilities/
    GA_Primary.uasset
    GA_Secondary.uasset
    GA_Teriary.uasset
```

`/Game/...` 是 UE 资源路径，对应磁盘上的 `Content/...`。GameMode 的配置路径以 `_C` 结尾，指向蓝图生成类；不要将它当作另一个磁盘文件。

玩家蓝图目前在 `Characters/Player/BP_GAS_PlayerCharacter.uasset`，原笔记的 `Characters/BP_GAS_PlayerCharacter.uasset` 目录已过时。文件位置已确认，但资源重定位后的引用和蓝图父类需在编辑器核验。

## 新增 C++ 入口

以下路径以 `Source/GAS_Acquaintance` 为根，每项均有同名 `.h` / `.cpp`：

| 路径（不含扩展名） | 对应笔记 |
| --- | --- |
| `Characters/Enemy/GAS_EnemyCharacter` | [敌人角色](../Characters/EnemyCharacter.md) |
| `AbilitySystem/GAS_AbilitySystemComponent` | [自定义 ASC](../AbilitySystem/AbilitySystemComponent.md) |
| `AbilitySystem/Abilities/GAS_GameplayAbility` | [能力基类](../AbilitySystem/GameplayAbility.md) |

事件标签仍在 `GameplayTags/GAS_Tags` 中定义，当前没有自定义 C++ AnimNotify；通知和动画文件的存在及待核验关系见 [Gameplay Event](../AbilitySystem/GameplayEvents.md)。

## 在编辑器中的核对顺序

```mermaid
flowchart TD
    map["打开 Map_Main：核对 World Settings 的 GameMode 覆盖"] -->|"明确当前使用的 GameMode 后"| classes["核对 Pawn / Controller / PlayerState 类"]
    classes -->|"确认玩家对象类型后"| input["核对 Controller 的两个 IMC、六个 Action 和二维输入类型"]
    input -->|"输入引用确认后"| startup["核对角色 GAS Abilities 中的 StartupAbilities 数组"]
    startup -->|"确认授予的能力类后"| assets["核对能力 Tag、激活与结束逻辑、网络设置"]
    assets -->|"配置核对完成后运行"| movement["验证移动、观察、跳跃"]
    movement -->|"基础输入正常后"| ability["验证三个能力输入"]
    ability -->|"涉及联网行为时"| network["分别观察服务端与客户端"]
    network -->|"验证敌人分支时"| enemy["核对敌人父类、启动数组、BeginPlay 和复制设置"]
    enemy -->|"涉及动画事件时"| events["核对动画引用、通知轨道、事件 Tag、目标与接收逻辑"]
```

该图是建议的人工核对顺序，不是引擎自动执行流程。

以上是复现和验证步骤，本次文档整理没有执行 UE 编辑器运行验证。
