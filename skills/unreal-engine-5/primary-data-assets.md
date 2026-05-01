---
name: primary-data-assets
description: Utilizing UPrimaryDataAsset and the AssetManager for asynchronous database loading. Replaces hard-references for Game Items.
---

# Primary Data Assets (UE5 Expert)

## Activation / When to Use
- Mandatory when defining complex objects (Weapons, Characters, Items, Skills) that need to be queried from a database without loading them all into memory.
- Trigger when transitioning away from DataTables for objects that hold complex pointers (Mesh, Audio, Particles).

## 1. Core Principles
A `UPrimaryDataAsset` is tracked by the global `UAssetManager`.
- **Queryable**: You can ask the AssetManager for a list of all Weapons without actually loading the weapon data into RAM.
- **Asynchronous**: You can tell the AssetManager to load a specific DataAsset into RAM only when the player equips it.

## 2. Implementation Syntax

### The C++ Class
Inherit from `UPrimaryDataAsset` and override `GetPrimaryAssetId`.

```cpp
#pragma once
#include "Engine/DataAsset.h"
#include "MyItemData.generated.h"

UCLASS()
class MYPROJECT_API UMyItemData : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Item")
    FName ItemID;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Item")
    FText DisplayName;

    // Use Soft Pointers so the Data Asset doesn't force-load the 3D model!
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Visuals")
    TSoftObjectPtr<UStaticMesh> ItemMesh;

    // REQUIRED OVERRIDE FOR ASSET MANAGER
    virtual FPrimaryAssetId GetPrimaryAssetId() const override
    {
        return FPrimaryAssetId(FName("ItemData"), GetFName());
    }
};
```

## 3. Project Settings Setup
To make the AssetManager aware of your class:
1. Go to Project Settings -> Asset Manager.
2. Add a new `Primary Asset Type`.
3. Name: `ItemData` (Must match the FName in `GetPrimaryAssetId`).
4. Base Class: `UMyItemData`.
5. Directories: `/Game/Data/Items` (Where the BP assets will be saved).

## 4. Querying and Loading via C++
```cpp
#include "Engine/AssetManager.h"

void AMySystem::LoadItem(FName ItemAssetName)
{
    UAssetManager& Manager = UAssetManager::Get();
    FPrimaryAssetId AssetId(FName("ItemData"), ItemAssetName);

    // 1. Get info without loading the asset
    FAssetData AssetData;
    Manager.GetPrimaryAssetData(AssetId, AssetData);

    // 2. Load it asynchronously
    TArray<FName> Bundles; // Optional specific load states
    Manager.LoadPrimaryAsset(AssetId, Bundles, FStreamableDelegate::CreateLambda([this, AssetId]()
    {
        UAssetManager& Mgr = UAssetManager::Get();
        if (UMyItemData* LoadedItem = Cast<UMyItemData>(Mgr.GetPrimaryAssetObject(AssetId)))
        {
            // Use the loaded item data!
        }
    }));
}
```

## 5. Impact on Safety
- **Memory Optimization**: By using `TSoftObjectPtr` inside a `UPrimaryDataAsset`, and then loading the DataAsset asynchronously, you create an incredibly memory-safe architecture for RPGs with thousands of items.
- **Scalability**: Unlike DataTables (which load everything into RAM at once), Primary Data Assets scale infinitely.

## Verification Checklist
- [ ] Class inherits from `UPrimaryDataAsset`.
- [ ] `GetPrimaryAssetId()` is overridden.
- [ ] Assets are registered in the Asset Manager project settings.
- [ ] Internal heavy assets (Meshes, Sounds) use `TSoftObjectPtr`.
