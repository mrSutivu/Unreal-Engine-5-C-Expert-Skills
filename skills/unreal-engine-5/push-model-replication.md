---
name: push-model-replication
description: Integrating UE5 Push Model (MARK_PROPERTY_DIRTY) for high-performance replication. Bypasses the traditional polling overhead.
---

# Push Model Replication (UE5 Expert)

## Activation / When to Use
- Mandatory when optimizing large multiplayer games (e.g., Battle Royales, MMOs).
- Trigger when you have thousands of replicated variables (e.g., 100 players * 50 inventory slots) and the server CPU is bottlenecked by replication polling.

## 1. The Polling Problem
By default, Unreal Engine iterates through EVERY replicated variable on EVERY replicated actor, every `NetUpdateFrequency`, to check if the value changed. This O(N) checking destroys server CPU.

## 2. Push Model (The Solution)
Push Model allows the developer to tell the Engine: "I just changed this variable. Mark it for replication." The engine then skips polling all other variables.

### Step 1: Enable in Build.cs
```csharp
PublicDependencyModuleNames.AddRange(new string[] { "NetCore" });
```
Also ensure `PushModel` is enabled in `DefaultEngine.ini`.

### Step 2: Header Declaration
Wrap the variable in a `ReplicatedUsing` (optional) but use the `F_` macros for Push Model if available, or just standard `UPROPERTY`.

```cpp
#pragma once
#include "Net/UnrealNetwork.h"

UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()
private:
    UPROPERTY(Replicated)
    int32 ResourceCount;

public:
    void SetResourceCount(int32 NewCount);
};
```

### Step 3: Source Implementation
You must use `MARK_PROPERTY_DIRTY_FROM_NAME` whenever you change the value.

```cpp
#include "Net/Core/PushModel/PushModel.h"

void AMyActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // FDoRepLifetime is the Push Model optimized version of DOREPLIFETIME
    FDoRepLifetimeParams Params;
    Params.bIsPushBased = true;
    DOREPLIFETIME_WITH_PARAMS_FAST(AMyActor, ResourceCount, Params);
}

void AMyActor::SetResourceCount(int32 NewCount)
{
    if (HasAuthority() && ResourceCount != NewCount)
    {
        ResourceCount = NewCount;
        
        // Notify the Engine that this specific property needs to be replicated
        MARK_PROPERTY_DIRTY_FROM_NAME(AMyActor, ResourceCount, this);
    }
}
```

## 3. Impact on Safety & Architecture
- **Setter Enforcement**: You MUST encapsulate replicated variables behind Setter functions. If another class does `Actor->ResourceCount = 5;` without calling `MARK_PROPERTY_DIRTY_FROM_NAME`, the change will NEVER replicate.
- **CPU Savings**: Drops replication overhead by up to 90% in dense scenes.

## Common Mistakes (BAD vs GOOD)

**BAD (Direct Access)**:
```cpp
// Inside another class
MyActor->ResourceCount = 10; // BAD: Push model is unaware. Client will never see '10'.
```

**GOOD (Setter with Push)**:
```cpp
MyActor->SetResourceCount(10); // Inside: MARK_PROPERTY_DIRTY_FROM_NAME is called.
```

## Verification Checklist
- [ ] `NetCore` is added to `Build.cs`.
- [ ] `DOREPLIFETIME_WITH_PARAMS_FAST` with `bIsPushBased = true` is used in `GetLifetimeReplicatedProps`.
- [ ] Variables are modified strictly through Setters.
- [ ] `MARK_PROPERTY_DIRTY_FROM_NAME` is called immediately after value modification.
