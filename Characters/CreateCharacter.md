# 创建角色基类

创建继承自 `ACharacter` 的角色基类，集中管理各类角色的公共功能，供具体角色子类复用。

使用 `UCLASS(Abstract)` 将其标记为抽象类，避免直接在场景中放置该基类；实际角色应使用具体子类。

```cpp
UCLASS(Abstract)
class GAS_ACQUAINTANCE_API AGAS_BaseCharacters : public ACharacter
{
    GENERATED_BODY()

public:
    AGAS_BaseCharacters();
};
```
