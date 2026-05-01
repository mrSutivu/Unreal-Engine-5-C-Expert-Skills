---
name: property-replication-doreplifetime
description: Utilizing GetLifetimeReplicatedProps and DOREPLIFETIME to synchronize variable state across the network.
---

# Property Replication & DOREPLIFETIME (UE5 Expert)

## Activation / When to Use
- Mandatory when data must be synchronized automatically from the Server to Clients.
- Trigger when creating health, ammo, score, or transform state variables.

## 1. Core Principles
- **One-Way Traffic**: Variables ONLY replicate from Server -> Client. A Client modifying a replicated variable only changes its local copy (which will be overwritten by the Server shortly after).
- **Polling**: By default, Unreal polls replicated variables every `NetUpdateFrequency` to see if they changed.

## 2. Implementation Syntax

### Header (.h)
Variables must be marked `Replicated` or `ReplicatedUsing`.
```cpp
#pragma once
#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyActor.generated.h"

UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()

protected:
    UPROPERTY(Replicated)
    int32 Health;

    UPROPERTY(Replicated)
    FVector TargetLocation;
};
```

### Source (.cpp)
You MUST override `GetLifetimeReplicatedProps`.
```cpp
#include "MyActor.h"
// Required for DOREPLIFETIME macros
#include "Net/UnrealNetwork.h" 

void AMyActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // Standard Replication (to everyone)
    DOREPLIFETIME(AMyActor, Health);
    DOREPLIFETIME(AMyActor, TargetLocation);
}
```

## 3. Optimization Conditions (COND_*)
Replicating everything to everyone wastes bandwidth. Use conditions to restrict replication.

- `COND_OwnerOnly`: Replicates ONLY to the Client that owns the Actor (e.g., Inventory Ammo, Input Data).
- `COND_SkipOwner`: Replicates to everyone EXCEPT the owner (e.g., if the owner does client-side prediction for visual rotation).
- `COND_InitialOnly`: Replicates once when the Actor spawns, then never again (e.g., a static Color ID).

```cpp
void AMyActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // Only the owning player needs to know their exact exact stamina value every frame
    DOREPLIFETIME_CONDITION(AMyActor, Stamina, COND_OwnerOnly);

    // Replicate team color once, it never changes
    DOREPLIFETIME_CONDITION(AMyActor, TeamColor, COND_InitialOnly);
}
```

## 4. Impact on Safety
- **Bandwidth Saturation**: Without conditions (`COND_OwnerOnly`), private data broadcasts to 99 players who don't need it.
- **Linker Errors**: Forgetting `#include "Net/UnrealNetwork.h"` is the #1 cause of compile failures when adding replication.

## Common Mistakes (BAD vs GOOD)

**BAD (Client-Side State Change)**:
```cpp
// Client attempts to heal itself
void AMyCharacter::DrinkPotion() {
    Health += 50; // BAD: The Server's value (say, 10) will overwrite this back to 10 in a split second.
}
```

**GOOD (Server-Driven State)**:
```cpp
void AMyCharacter::DrinkPotion() {
    if (HasAuthority()) { Health += 50; } // Server updates, automatically replicates to client.
    else { Server_RequestDrinkPotion(); } // Client asks server to do it.
}
```

## Verification Checklist
- [ ] Variable has `UPROPERTY(Replicated)`.
- [ ] `GetLifetimeReplicatedProps` is overridden.
- [ ] `Super::GetLifetimeReplicatedProps(OutLifetimeProps)` is called inside.
- [ ] `#include "Net/UnrealNetwork.h"` is present in the `.cpp`.
- [ ] `DOREPLIFETIME_CONDITION` is used to filter unnecessary network traffic.
