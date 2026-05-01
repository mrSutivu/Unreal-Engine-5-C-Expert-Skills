---
name: programming-subsystems-architecture
description: Implementation of UE5 Subsystems (GameInstance, World, LocalPlayer) to replace monolithic singleton patterns.
---

# Programming Subsystems Architecture (UE5 Expert)

## Activation / When to Use
- Mandatory when creating manager classes (e.g., Quest Manager, Sound Manager, Economy Manager).
- Trigger when refactoring "God Classes" (like massive `UGameInstance` or `AGameMode` blueprints).
- Replaces custom Singletons.

## 1. Core Principles
Subsystems are auto-instantiated and auto-destroyed by the engine based on their specific lifecycle.
- **Zero Setup**: You do not need to spawn them, place them in a level, or initialize them manually.
- **C++ and BP Support**: Fully accessible globally in Blueprints.

## 2. The 5 Subsystem Types

1. **`UEngineSubsystem`**: Lives as long as the Editor/Game executable.
2. **`UEditorSubsystem`**: Editor-only. Good for tooling.
3. **`UGameInstanceSubsystem`**: Lives as long as the game is running. Persists across level loads. (Best for Profile/Save data).
4. **`UWorldSubsystem`**: Tied to the Level (`UWorld`). Destroyed when the level unloads. (Best for Quest/Spawner managers).
5. **`ULocalPlayerSubsystem`**: Tied to a specific local screen/player. (Best for split-screen UI managers).

## 3. Implementation Syntax

### Header (.h)
```cpp
#pragma once
#include "CoreMinimal.h"
#include "Subsystems/WorldSubsystem.h"
#include "MyQuestSubsystem.generated.h"

UCLASS()
class MYPROJECT_API UMyQuestSubsystem : public UWorldSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    UFUNCTION(BlueprintCallable, Category = "Quests")
    void CompleteQuest(FName QuestID);
};
```

### C++ Access
```cpp
void AMyActor::FinishLevel()
{
    if (UWorld* World = GetWorld())
    {
        if (UMyQuestSubsystem* QuestSys = World->GetSubsystem<UMyQuestSubsystem>())
        {
            QuestSys->CompleteQuest("Q_01");
        }
    }
}
```

## 4. Impact on Safety
- **Memory Leaks Eliminated**: Custom singletons often leak memory if not torn down correctly. Subsystems guarantee deterministic destruction.
- **Dependency Injection**: Subsystems can depend on other subsystems by accessing the `FSubsystemCollectionBase` during `Initialize()`.

## Common Mistakes (BAD vs GOOD)

**BAD (Custom Singleton Pattern)**:
```cpp
class FMyAudioSingleton {
    static FMyAudioSingleton* Instance;
    // DANGEROUS: Must handle GC, World switches, and PIE (Play-In-Editor) resets manually.
};
```

**GOOD (UE5 Subsystem)**:
```cpp
UCLASS()
class UMyAudioSubsystem : public UGameInstanceSubsystem {
    // SAFE: Engine handles creation, GC, and teardown automatically.
};
```

## Verification Checklist
- [ ] Appropriate Subsystem base class is chosen based on required lifespan.
- [ ] No manual instantiation (`NewObject`) is used for Subsystems.
- [ ] Overrides for `Initialize` and `Deinitialize` are used for setup/teardown instead of constructors.
- [ ] Replicated data is NOT stored here (Subsystems do not replicate natively; use GameState for that).
