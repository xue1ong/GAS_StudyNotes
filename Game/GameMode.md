# GameMode：游戏框架的配置入口

GameMode 负责游戏规则和玩家对象的创建配置；联网游戏中的 GameMode 位于服务端。客户端需要使用的玩家状态，应通过 PlayerState 等可复制对象提供。

本项目没有自定义 GameMode C++ 类，业务目录存在蓝图资产：

```text
Content/GAS_Acquaintance/Blueprints/Game/BP_GAS_Gamemode.uasset
```

## 已确认的项目配置

`Config/DefaultEngine.ini` 的 `GameMapsSettings` 段指定：

```ini
GameDefaultMap=/Game/GAS_Acquaintance/Maps/Map_Main.Map_Main
EditorStartupMap=/Game/GAS_Acquaintance/Maps/Map_Main.Map_Main
GlobalDefaultGameMode=/Game/GAS_Acquaintance/Blueprints/Game/BP_GAS_Gamemode.BP_GAS_Gamemode_C
```

即项目默认运行地图和编辑器启动地图都是 `Map_Main`，项目默认 GameMode 为 `BP_GAS_Gamemode`。具体地图的 World Settings 还可能覆盖 GameMode，运行时应检查该项；本次未核验地图内部设置。

## 在蓝图默认值中核对的关系

| GameMode 属性 | 对应资源 | 核对原因 |
| --- | --- | --- |
| Default Pawn Class | `BP_GAS_PlayerCharacter` | 让实际生成的角色具有 ASC 访问与初始化逻辑 |
| Player Controller Class | `BP_GAS_PlayerController` | 使用项目的 Enhanced Input 和能力激活入口 |
| Player State Class | `BP_GAS_PlayerState` | 提供角色所需的 ASC |

以上蓝图文件均已确认存在，但表格是**编辑器核对清单**，不是已读取到的蓝图属性值。仅有 `.ini` 中的 GameMode 路径，无法证明这三个类已正确赋值。

```mermaid
flowchart TD
    settings["DefaultEngine.ini"] -->|"GlobalDefaultGameMode 指定"| mode["BP_GAS_Gamemode"]
    settings -->|"默认游戏地图和编辑器启动地图指定"| map["Map_Main"]
    map -.->|"World Settings 可能覆盖，待核验"| mode
    mode -.->|"Default Pawn Class 预期配置，待核验"| pawn["BP_GAS_PlayerCharacter"]
    mode -.->|"Player Controller Class 预期配置，待核验"| controller["BP_GAS_PlayerController"]
    mode -.->|"Player State Class 预期配置，待核验"| state["BP_GAS_PlayerState"]
    pawn -->|"运行时 getter 需要兼容的 PlayerState 类型"| state
    state -->|"其 C++ 实现创建并提供"| asc["ASC"]
```

实线表示已确认的配置或 C++ 依赖，虚线表示尚需在编辑器核验的配置关系。

## 与 GAS 初始化的关系

本项目启动能力的授予入口在角色中，与 GameMode 的类配置通过以下依赖接起来。

```mermaid
flowchart TD
    config["GameMode 玩家类配置正确"] -.->|"运行前提，蓝图配置待核验"| possessed["角色被占有：PossessedBy"]
    possessed -->|"调用角色 getter"| getter["取得 PlayerState.ASC"]
    getter -->|"ASC 有效且服务端拥有权限"| info["InitAbilityActorInfo"]
    info -->|"随后调用"| startup["GiveStartupAbilities"]
    startup -->|"为每个启动能力类构造 Spec"| grant["ASC.GiveAbility"]
```

若 GameMode 选用了不兼容的 PlayerState，角色中的 `Cast<AGAS_PlayerState>` 会失败，后续初始化提前返回。排查技能无响应时，应先核对这些基础对象的实际类型。

参见：[项目配置](ProjectSetup.md)、[PlayerState](../Player/PlayerState.md)、[玩家角色](../Characters/PlayerCharacter.md)。
