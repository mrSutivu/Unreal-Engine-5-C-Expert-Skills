---
name: async-loading-tsoftobjectptr
description: Best practices for UE 5.4 Async Loading using TSoftObjectPtr and FStreamableManager. Prevents memory bloat and asset loading cascades.
---

# Async Loading with TSoftObjectPtr (UE5 Expert)

## Activation / When to Use
- Mandatory for high-memory assets (Skeletal Meshes, High-res Textures, Audio).
- Trigger when spawning data-heavy actors dynamically during gameplay.
- DO NOT use hard pointers (`TObjectPtr<UStaticMesh>`) unless the asset MUST be loaded in RAM 100% of the time.

## 1. The Hard vs Soft Pointer Problem
A Hard Pointer (`TObjectPtr<UTexture2D>`) forces the Engine to load the Texture into memory the moment the Actor class is loaded. This causes "Loading Cascades" (loading one UI widget loads all item icons in the game).
A Soft Pointer (`TSoftObjectPtr<UTexture2D>`) only stores a *String Path* to the asset. It takes 0 memory until explicitly loaded.

## 2. Declaration Syntax

```cpp
// Header
UPROPERTY(EditAnywhere, Category = "Assets")
TSoftObjectPtr<USkeletalMesh> HeavyMeshRef;

UPROPERTY(EditAnywhere, Category = "Assets")
TSoftClassPtr<AEnemy> EnemyClassRef;

// Handle to keep asset alive in memory
TSharedPtr<FStreamableHandle> AssetHandle;
```

## 3. UE 5.4 Async Loading Implementation
Use the global `FStreamableManager` provided by the `UAssetManager`.

```cpp
#include "Engine/AssetManager.h"
#include "Engine/StreamableManager.h"

void AMySpawner::LoadEnemyAsync()
{
    if (EnemyClassRef.IsNull()) return;

    FStreamableManager& Streamable = UAssetManager::Get().GetStreamableManager();

    // Request Load with Lambda Callback
    AssetHandle = Streamable.RequestAsyncLoad(EnemyClassRef.ToSoftObjectPath(), FStreamableDelegate::CreateLambda([this]()
    {
        if (EnemyClassRef.IsValid())
        {
            UClass* LoadedClass = EnemyClassRef.Get();
            GetWorld()->SpawnActor<AEnemy>(LoadedClass, GetActorTransform());
        }
    }));
}
```

## 4. Memory Management (The Handle)
- The `AssetHandle` acts as a GC lock. As long as the Handle exists, the loaded asset stays in RAM.
- **Unloading**: To free memory, simply release the handle: `AssetHandle.Reset()`. The GC will clean up the asset on its next pass.

## 5. Impact on Safety
- **Memory Bloat**: Prevents Out-Of-Memory (OOM) crashes on consoles by ensuring agents only load necessary Artifacts.
- **Hitch Prevention**: Async loading happens on a background thread. Forcing synchronous loads (e.g., `.LoadSynchronous()`) on the Game Thread causes frame drops.

## Common Mistakes (BAD vs GOOD)

**BAD (Sync Load on Game Thread)**:
```cpp
USkeletalMesh* Mesh = HeavyMeshRef.LoadSynchronous(); // BLOCKS THE GAME THREAD!
```

**GOOD (Async Load)**:
```cpp
Streamable.RequestAsyncLoad(HeavyMeshRef.ToSoftObjectPath(), Delegate); // SAFE, No hitch.
```

**BAD (Hard Pointer Cascade)**:
```cpp
UPROPERTY(EditAnywhere)
UTexture2D* Icon; // Loads Icon into memory immediately.
```

**GOOD (Soft Pointer deferred)**:
```cpp
UPROPERTY(EditAnywhere)
TSoftObjectPtr<UTexture2D> Icon; // Takes 0 memory until needed.
```

## Verification Checklist
- [ ] Heavy assets use `TSoftObjectPtr` or `TSoftClassPtr`.
- [ ] `UAssetManager::Get().GetStreamableManager()` is used for async requests.
- [ ] `LoadSynchronous()` is avoided in gameplay code.
- [ ] A `TSharedPtr<FStreamableHandle>` is kept if the asset needs to remain in memory.
- [ ] Validity (`IsValid()`) is checked inside the async callback before using `.Get()`.
