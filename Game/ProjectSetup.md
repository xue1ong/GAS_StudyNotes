# 配置文件、构建类与资源索引

本页按实际文件归档非玩法类内容。参考项目为 `..\GAS_Learning`，核对日期：2026-11-09。

## GAS_Learning.uproject

引擎关联为 `5.6`；运行时模块为 `GAS_Learning`，加载阶段 Default。`GameplayAbilities` 插件已启用，`ModelingToolsEditorMode` 仅面向 Editor 启用。

## GAS_Learning.Build.cs：GAS_Learning

构建规则类继承 `ModuleRules`。构造函数设置 `PCHUsageMode.UseExplicitOrSharedPCHs`；公开依赖为 Core、CoreUObject、Engine、InputCore、EnhancedInput、GameplayAbilities、GameplayTags、GameplayTasks、UMG，私有依赖数组为空。

| 模块 | 项目中的使用者 |
| --- | --- |
| EnhancedInput | AGAS_PlayerController 的输入 Action、Mapping Context 和子系统 API |
| GameplayAbilities | 自定义 ASC、能力类、GAS 接口和 Spec |
| GameplayTags | GAS_Tags 命名空间及 Tag 查询 |
| GameplayTasks | GAS 相关依赖；当前没有自定义 Task 类 |
| UMG | UGAS_WidgetComponent 继承的 UWidgetComponent 及 UI 类型依赖 |

## Target.cs：游戏与编辑器目标类

| 文件 / 类 | 基类 | 构造设置 |
| --- | --- | --- |
| `Source/GAS_Learning.Target.cs` / GAS_LearningTarget | TargetRules | TargetType.Game |
| `Source/GAS_LearningEditor.Target.cs` / GAS_LearningEditorTarget | TargetRules | TargetType.Editor |

两个构造函数均使用 BuildSettingsVersion.V5、EngineIncludeOrderVersion.Unreal5_6，并将 `GAS_Learning` 加入 ExtraModuleNames。

## GAS_Learning.h / .cpp：模块入口

头文件只包含 `CoreMinimal.h`，不定义玩法类；源文件通过 `IMPLEMENT_PRIMARY_GAME_MODULE(FDefaultGameModuleImpl, GAS_Learning, "GAS_Learning")` 注册主游戏模块。

## Config：地图、GameMode 与输入类型

`DefaultEngine.ini` 的 GameMapsSettings 指定：

```ini
GameDefaultMap=/Game/GAS_Learning/Maps/Map_Main.Map_Main
EditorStartupMap=/Game/GAS_Learning/Maps/Map_Main.Map_Main
GlobalDefaultGameMode=/Game/GAS_Learning/Blueprints/Game/BP_GAS_Gamemode.BP_GAS_Gamemode_C
```

`DefaultInput.ini` 指定：

```ini
DefaultPlayerInputClass=/Script/EnhancedInput.EnhancedPlayerInput
DefaultInputComponentClass=/Script/EnhancedInput.EnhancedInputComponent
```

```mermaid
flowchart TD
    project["uproject"] -->|"启用 GameplayAbilities 插件"| plugin["GAS 插件模块"]
    build["Build.cs：公开依赖"] -->|"声明可使用的 GAS API"| gas["ASC、能力类与原生标签"]
    plugin -->|"提供实现"| gas
    build -->|"依赖 EnhancedInput"| controller["AGAS_PlayerController 输入绑定"]
    input["DefaultInput.ini"] -->|"选择输入默认类"| controller
    engine["DefaultEngine.ini"] -->|"指定启动地图"| map["Map_Main"]
    engine -->|"指定全局默认生成类"| mode["BP_GAS_Gamemode"]
    map -.->|"地图级 GameMode 覆盖，待核验"| mode
```

图中是配置依赖，不表示运行时调用顺序。GameMode 类配置见 [BP_GAS_Gamemode](GameMode.md)。

## 脚本模块重定向与资源路径

当前源码 API 宏为 `GAS_LEARNING_API`，头文件包含路径以 `GAS_Learning/...` 为根。`DefaultEngine.ini` 保留下面的脚本包重定向：

```ini
[CoreRedirects]
+PackageRedirects=(OldName="/Script/GAS_Acquaintance",NewName="/Script/GAS_Learning")
```

模块、业务资源、默认地图与 GameMode 均使用 GAS_Learning。上述配置为旧脚本包引用提供重定向。

## Content：业务资产

以下路径以 `Content/GAS_Learning` 为根：

```text
Maps/Map_Main.umap
UI/
  BP_PlayerStatsWidget.uasset
  BP_ManaBar.uasset
  BP_HealthBar.uasset
  BP_GAS_WidgetComponent.uasset
Blueprints/
  Game/BP_GAS_Gamemode.uasset
  Player/BP_GAS_PlayerController.uasset
  Player/BP_GAS_PlayerState.uasset
  Characters/
    Player/
      BP_GAS_PlayerCharacter.uasset
      Anim/AM_GAS_Primary_A.uasset
      Anim/AM_GAS_Primary_B.uasset
      Anim/Primary_A.uasset
      Anim/Primary_B.uasset
    Enemy/
      BP_GAS_EnemyBase.uasset
      BP_Enemy_Melee.uasset
      BP_Enemy_Ranged.uasset
      Anim/ABP_Enemy_Melee.uasset
      Anim/ABP_Enemy_Ranged.uasset
      Anim/AM_HitReact_Melee.uasset
      Anim/AM_HitReact_Ranged.uasset
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
    AbilityActions/IA_LevelUpAbility.uasset
  AbilitySystem/
    GameplayEffects/GE_Initialize_Attributes.uasset
    GameplayEffects/Player/GE_PlayerDamage.uasset
    GameplayEffects/Player/CT_PlayerDamage.uasset
    PlayerAbilities/GA_GAS_Primary.uasset
    PlayerAbilities/GA_Secondary.uasset
    PlayerAbilities/GA_Teriary.uasset
    EnemyAbilities/GA_GAS_HitReact.uasset
    EnemyAbilities/GA_GAS_HitReact_Melee.uasset
    EnemyAbilities/GA_GAS_HitReact_Ranged.uasset
```

`/Game/...` 对应磁盘 `Content/...`；GameMode 路径的 `_C` 指蓝图生成类。上述文件存在已核对，蓝图父类、默认引用、动画通知轨道和能力事件图没有经过本次编辑器核验。

玩法类的完整文件索引见 [README](../README.md)，成员与函数说明保留在各类笔记中。

初始化效果、伤害效果与曲线、升级输入和四个 UI 资产均有对应磁盘文件。效果数值与引用、升级 Action 的蓝图绑定、UI 父类和挂载待编辑器核验。
