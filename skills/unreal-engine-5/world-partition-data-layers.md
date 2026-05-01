---
name: world-partition-data-layers
description: Interacting with World Partition and Data Layers in C++ to load/unload regions dynamically in Open Worlds.
---

# World Partition & Data Layers (UE5 Expert)

## Activation / When to Use
- Mandatory when building Open World games.
- Trigger when you need to load/unload specific scenarios, quests, or alternative world states (e.g., a destroyed village vs a peaceful village) dynamically.

## 1. Core Principles
World Partition divides the map into a grid and loads it automatically based on the player's position.
- **Data Layers**: An override system. You can assign Actors to a "Data Layer" (e.g., "Quest_01_Actors"). You can then activate this layer via C++ to load all those actors instantly, regardless of the grid.

## 2. Implementation Syntax

### Activating a Data Layer (C++)
Use the `UDataLayerManager` available via the `UWorld`.

```cpp
#include "WorldPartition/DataLayer/DataLayerManager.h"
#include "WorldPartition/DataLayer/DataLayerAsset.h"

void AQuestManager::ActivateQuestLayer(UDataLayerAsset* QuestLayer)
{
    if (UWorld* World = GetWorld())
    {
        if (UDataLayerManager* LayerManager = UDataLayerManager::GetDataLayerManager(World))
        {
            // Activate the layer (Loads the actors into memory and makes them visible)
            LayerManager->SetDataLayerInstanceRuntimeState(
                QuestLayer, 
                EDataLayerRuntimeState::Activated, 
                /*bRecursive*/ true
            );
        }
    }
}
```

### The Runtime States
- `Unloaded`: Actors consume 0 memory.
- `Loaded`: Actors are in memory, but hidden and do not tick. (Good for pre-loading a boss fight).
- `Activated`: Actors are visible and ticking.

## 3. Streaming Sources (Manual Loading)
Sometimes you need to force the grid to load at a specific location (e.g., the destination of a teleport before the player arrives).

```cpp
#include "WorldPartition/WorldPartitionSubsystem.h"

void ATeleportManager::PreloadDestination(FVector TargetLocation)
{
    if (UWorldPartitionSubsystem* WPSubsystem = GetWorld()->GetSubsystem<UWorldPartitionSubsystem>())
    {
        // Add a temporary streaming source to force the grid to load around TargetLocation
        FWorldPartitionStreamingSource Source;
        Source.Name = FName("TeleportPreload");
        Source.Location = TargetLocation;
        Source.Rotation = FRotator::ZeroRotator;
        Source.TargetState = EStreamingSourceTargetState::Activated;
        Source.TargetBehavior = EStreamingSourceTargetBehavior::Include;

        // NOTE: Managing sources manually requires keeping track of them or using a specialized component like UWorldPartitionStreamingSourceComponent
    }
}
```

## 4. Impact on Safety
- **Hard References**: If an Actor in an Unloaded Data Layer holds a hard pointer (`UPROPERTY() AActor*`) to an Actor in the base grid, it can cause GC issues or crashes when layers transition.
- **IsSpatialLoaded**: World Partition Actors implement `bIsSpatiallyLoaded`. If false, they are ALWAYS loaded regardless of the grid. Agents must ensure generic props are set to `true`.

## Verification Checklist
- [ ] `UDataLayerManager` is used to toggle Data Layer states.
- [ ] States step logically (`Unloaded` -> `Loaded` -> `Activated`) to hide streaming hitches.
- [ ] `UWorldPartitionStreamingSourceComponent` is attached to any non-player actor that needs the world to load around it (like an AI traversing the map).
