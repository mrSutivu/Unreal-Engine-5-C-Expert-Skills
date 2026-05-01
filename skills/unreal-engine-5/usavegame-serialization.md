---
name: usavegame-serialization
description: Writing and reading persistent game data using USaveGame objects and UGameplayStatics::SaveGameToSlot.
---

# USaveGame Serialization (UE5 Expert)

## Activation / When to Use
- Mandatory for persistent player progression (Levels, Inventory, Settings).
- Trigger when data needs to survive closing and reopening the game executable.

## 1. Core Principles
The `USaveGame` class is a data container. The engine handles the low-level binary serialization of any `UPROPERTY` defined within it.

## 2. Implementation Syntax

### The SaveGame Class (.h)
```cpp
#pragma once
#include "GameFramework/SaveGame.h"
#include "MySaveGame.generated.h"

UCLASS()
class MYPROJECT_API UMySaveGame : public USaveGame
{
    GENERATED_BODY()

public:
    UPROPERTY(VisibleAnywhere, Category = "Basic")
    FString PlayerName;

    UPROPERTY(VisibleAnywhere, Category = "Basic")
    int32 PlayerLevel;
    
    // Can hold complex structs if they are marked USTRUCT()
    UPROPERTY(VisibleAnywhere, Category = "Inventory")
    TArray<int32> InventoryIDs;
};
```

### Saving Data (Async)
Never save on the Game Thread if the payload is large (causes hitches). Use `AsyncSaveGameToSlot`.

```cpp
void AMyManager::SaveGame()
{
    UMySaveGame* SaveInst = Cast<UMySaveGame>(UGameplayStatics::CreateSaveGameObject(UMySaveGame::StaticClass()));
    
    SaveInst->PlayerName = TEXT("Player1");
    SaveInst->PlayerLevel = 42;

    // FAsyncSaveGameToSlotDelegate
    FAsyncSaveGameToSlotDelegate SavedDelegate;
    SavedDelegate.BindUObject(this, &AMyManager::OnSaveFinished);

    UGameplayStatics::AsyncSaveGameToSlot(SaveInst, TEXT("Slot1"), 0, SavedDelegate);
}
```

### Loading Data (Async)
```cpp
void AMyManager::LoadGame()
{
    FAsyncLoadGameFromSlotDelegate LoadedDelegate;
    LoadedDelegate.BindUObject(this, &AMyManager::OnLoadFinished);

    UGameplayStatics::AsyncLoadGameFromSlot(TEXT("Slot1"), 0, LoadedDelegate);
}

void AMyManager::OnLoadFinished(const FString& SlotName, const int32 UserIndex, USaveGame* LoadedGameData)
{
    if (UMySaveGame* MyData = Cast<UMySaveGame>(LoadedGameData))
    {
        // Apply loaded data
        int32 Level = MyData->PlayerLevel;
    }
}
```

## 3. Impact on Safety
- **Versioning**: Always include a `SaveVersion` integer in your `USaveGame`. If you add new variables in an update, check the version during Load to apply default values to old save files.
- **Pointers**: You CANNOT save pointers (`AActor*`) to disk. You must save unique IDs (Guids or FNames) and reconstruct/spawn the actors when loading.

## Common Mistakes (BAD vs GOOD)

**BAD (Saving raw pointers)**:
```cpp
UPROPERTY()
AActor* EquippedWeapon; // FATAL: Memory address changes every time game runs. Will crash on load.
```

**GOOD (Saving IDs)**:
```cpp
UPROPERTY()
FName EquippedWeaponClassID; // SAFE: Use this ID to spawn the weapon on load.
```

## Verification Checklist
- [ ] Class inherits from `USaveGame`.
- [ ] Variables to be saved are marked `UPROPERTY()`.
- [ ] `AsyncSaveGameToSlot` is used to prevent framerate hitches.
- [ ] No raw memory pointers (`UObject*`, `AActor*`) are stored in the save file.
