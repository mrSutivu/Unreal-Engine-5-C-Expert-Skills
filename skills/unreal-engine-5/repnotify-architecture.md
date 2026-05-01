---
name: repnotify-architecture
description: Using ReplicatedUsing to trigger logic on clients when a replicated variable changes. Handles OldValue comparisons.
---

# RepNotify Architecture (UE5 Expert)

## Activation / When to Use
- Mandatory when a Client needs to react instantly to a state change driven by the Server (e.g., playing a sound when Health reaches 0, changing a material when Team changes).
- Replaces the need for `NetMulticast` RPCs for state-driven visual updates.

## 1. Core Principles
A `RepNotify` is a UFUNCTION that is called automatically on the Client when it receives a new value for a replicated property from the Server.
- **State Reliability**: Unlike Multicast RPCs which can be dropped, a RepNotify guarantees the client eventually reaches the correct visual state, even if they join late (JIP) or experience lag.

## 2. Implementation Syntax

### Header (.h)
The variable must use `ReplicatedUsing = FunctionName`. The function must be a `UFUNCTION()`.
```cpp
UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()

protected:
    UPROPERTY(ReplicatedUsing = OnRep_CurrentState)
    EWeaponState CurrentState;

    // Must be a UFUNCTION
    UFUNCTION()
    virtual void OnRep_CurrentState();
};
```

### Source (.cpp)
Do not forget to register the variable in `GetLifetimeReplicatedProps`.
```cpp
void AMyActor::OnRep_CurrentState()
{
    // Update visuals based on the new value of CurrentState
    if (CurrentState == EWeaponState::Firing) {
        PlayFireAnimation();
    }
}
```

## 3. Passing the Old Value
In C++, `OnRep` functions can optionally take one parameter matching the property type. The engine will pass the *previous* value of the variable before the update.

```cpp
// Header
UFUNCTION()
void OnRep_Health(float OldHealth);

// Source
void AMyActor::OnRep_Health(float OldHealth)
{
    if (Health < OldHealth) {
        // Player took damage, play hit flash!
        PlayDamageFlash();
    }
}
```

## 4. Server Execution
**CRITICAL**: `OnRep` functions are NOT called automatically on the Server when the server changes the variable. You must call them manually if you want the server (as a listening host) to see the visual changes.

```cpp
void AMyActor::Server_SetState(EWeaponState NewState)
{
    if (HasAuthority())
    {
        CurrentState = NewState;
        
        // Listen Server doesn't trigger OnRep automatically. Call it manually.
        if (GetNetMode() != NM_DedicatedServer) {
            OnRep_CurrentState();
        }
    }
}
```

## 5. Impact on Safety
- **Late Joiners (JIP)**: If a barrel explodes via a Multicast RPC, a player joining 5 seconds later sees a whole barrel. If the barrel's state is `ReplicatedUsing=OnRep_Exploded`, the late joiner receives the `Exploded` state on join, triggers the `OnRep`, and correctly renders the destroyed barrel.

## Common Mistakes (BAD vs GOOD)

**BAD (Multicast for State)**:
```cpp
// If a client drops this packet, their door remains closed forever while the server thinks it's open.
Multicast_OpenDoor(); 
```

**GOOD (RepNotify for State)**:
```cpp
// State is guaranteed to sync. OnRep_bIsOpen handles the visual rotation.
bIsOpen = true; 
```

## Verification Checklist
- [ ] Variable uses `UPROPERTY(ReplicatedUsing = FuncName)`.
- [ ] The `OnRep` function is marked with `UFUNCTION()`.
- [ ] The variable is registered in `GetLifetimeReplicatedProps`.
- [ ] Server calls the `OnRep` function manually if it acts as a Listen Server.
