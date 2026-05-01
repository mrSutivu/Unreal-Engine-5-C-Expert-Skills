---
name: gameinstance-cross-level-state
description: Role of UGameInstance in cross-level persistence. Enforces separation between match-specific state and application-wide state.
---

# GameInstance Cross-Level State (UE5 Expert)

## Activation / When to Use
- Mandatory when data must survive level transitions (e.g., Map A to Map B).
- Trigger when storing application-wide configurations (e.g., Master Volume, Total Playtime, Profile Saves).

## 1. Core Principles
The `UGameInstance` is the highest-level manager in the game hierarchy.
- **Lifespan**: Created when the game executable launches. Destroyed when the game executable closes.
- **Persistence**: It outlives all Levels, `UWorld`s, `GameMode`s, and `PlayerController`s.
- **Networking**: It is STRICTLY LOCAL. It is never replicated over the network. The Server has its own GameInstance, and each Client has its own.

## 2. Responsibilities
- Managing network sessions (Create Session, Find Session, Join Session).
- Storing Cross-Map Data (e.g., "Player chose Team Red in the lobby, apply this when loading the actual map").
- Interacting with Online Subsystems (Steam, Epic Online Services).

## 3. Impact on Safety
- **Replication Confusion**: Agents often attempt to put multiplayer logic (like match scores) in the GameInstance. Because it doesn't replicate, this leads to massive desyncs. Use `GameState` for multiplayer data.
- **God Class Bloat**: In older versions of UE, GameInstance became a "God Class" holding 10,000 lines of code. In UE5, you must use `UGameInstanceSubsystem` to keep it modular (See Subsystems skill).

## Common Mistakes (BAD vs GOOD)

**BAD (Storing Match Data)**:
```cpp
UCLASS()
class UMyGameInstance : public UGameInstance {
    // BAD: This will not replicate to other players!
    UPROPERTY()
    int32 BlueTeamScore; 
};
```

**GOOD (Storing Local Profile Data)**:
```cpp
UCLASS()
class UMyGameInstance : public UGameInstance {
    // SAFE: Local graphics setting. Doesn't need replication.
    UPROPERTY()
    float FOVPreference; 
};
```

**BAD (Dangling Pointers)**:
```cpp
// Storing an Actor pointer in GameInstance
UPROPERTY()
AActor* LastInteractedActor; 
// CRASH: When the level changes, the Actor is destroyed, but the GameInstance outlives it. 
// The pointer becomes a garbage address.
```

**GOOD (Soft References across levels)**:
```cpp
UPROPERTY()
TSoftObjectPtr<AActor> LastInteractedActor; // Safe against garbage collection
```

## Verification Checklist
- [ ] `UGameInstance` contains NO multiplayer/replicated state.
- [ ] No hard pointers (`TObjectPtr<AActor>`) to level-specific objects are stored here.
- [ ] Custom `GameInstance` class is assigned in `Project Settings -> Maps & Modes`.
- [ ] Complex logic is offloaded to `UGameInstanceSubsystem` instead of bloating this class.
