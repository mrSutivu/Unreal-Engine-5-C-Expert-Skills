---
name: datatables-ftablerowbase
description: Defining structs with FTableRowBase. Looking up rows via C++ for secure data importation from CSV/JSON.
---

# DataTables & FTableRowBase (UE5 Expert)

## Activation / When to Use
- Mandatory when managing structured, static game data (e.g., Item Stats, Enemy Base Health, Dialogue lines, Crafting Recipes).
- Trigger when importing data from external sources (CSV/JSON) into the Engine.

## 1. Core Principles
A `UDataTable` is a grid of data. Every row shares the same structure. To create a DataTable in Unreal, you must first define a C++ `USTRUCT` that inherits from `FTableRowBase`.

## 2. Implementation Syntax

### The Row Structure
```cpp
#pragma once
#include "Engine/DataTable.h"
#include "MyItemStats.generated.h"

USTRUCT(BlueprintType)
struct FMyItemStats : public FTableRowBase
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    FText DisplayName;

    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    float BaseDamage;

    // Use Soft Pointers for assets to prevent loading the entire database into RAM!
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    TSoftObjectPtr<UTexture2D> Icon;
};
```

### Querying the DataTable (C++)
You usually store a pointer to the DataTable in a manager class.

```cpp
UPROPERTY(EditAnywhere, Category = "Database")
UDataTable* ItemDatabase;

void AMyManager::RetrieveItemData(FName RowName)
{
    if (ItemDatabase)
    {
        // Safe lookup. The ContextString is used for error logging.
        FString ContextString = TEXT("Item Retrieval Context");
        FMyItemStats* RowData = ItemDatabase->FindRow<FMyItemStats>(RowName, ContextString);

        if (RowData)
        {
            // Use data
            float Damage = RowData->BaseDamage;
        }
    }
}
```

## 3. Impact on Safety & Performance
- **RAM Explosion**: If your `FTableRowBase` struct uses hard pointers (`UTexture2D*`), loading the DataTable into memory will force-load EVERY texture for EVERY item simultaneously. **ALWAYS use `TSoftObjectPtr` for assets in DataTables.**
- **Validation**: When `FindRow` fails, it returns `nullptr` and logs the `ContextString`. This prevents silent data corruption and makes missing CSV entries obvious in the logs.

## Common Mistakes (BAD vs GOOD)

**BAD (Hard Assets)**:
```cpp
struct FEnemyData : public FTableRowBase {
    UPROPERTY() USkeletalMesh* Mesh; // BAD: 100 enemies = 100 meshes loaded at startup.
};
```

**GOOD (Soft Assets)**:
```cpp
struct FEnemyData : public FTableRowBase {
    UPROPERTY() TSoftObjectPtr<USkeletalMesh> Mesh; // SAFE: Meshes loaded on demand.
};
```

## Verification Checklist
- [ ] Struct inherits from `FTableRowBase`.
- [ ] Struct is marked `USTRUCT(BlueprintType)`.
- [ ] All asset references inside the struct use `TSoftObjectPtr` or `TSoftClassPtr`.
- [ ] `FindRow<T>` is used for retrieval, and the pointer is checked for null.
