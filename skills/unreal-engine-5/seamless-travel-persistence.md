---
name: seamless-travel-persistence
description: Managing player data and connections during level transitions. Usage of CopyProperties to persist PlayerStates across maps.
---

# Seamless Travel Persistence (UE5 Expert)

## Activation / When to Use
- Mandatory when moving players from a Lobby map to a Gameplay map, or between levels without dropping the network connection.
- Trigger when configuring `ServerTravel`.

## 1. Core Principles
- **Hard Travel (Default)**: Disconnects all clients, loads the new map, and reconnects them. Very slow, interrupts voice chat.
- **Seamless Travel**: Loads the destination map in the background (via a Transition Map) and moves the active connections over.

## 2. Enabling Seamless Travel
It must be explicitly enabled in the `GameMode`.
```cpp
AMyGameMode::AMyGameMode()
{
    bUseSeamlessTravel = true;
}
```
*Note: You must configure a "Transition Map" in Project Settings for this to work correctly.*

## 3. Persisting Data (The Actor Lifecycle)
During Seamless Travel, ALMOST EVERYTHING is destroyed, including Pawns, the GameMode, and the GameState. 

**What Survives?**
- `UGameInstance` (Always)
- `APlayerController` (By default)
- `APlayerState` (Requires manual configuration)

### Persisting PlayerState Data
When a new level loads, a NEW `PlayerState` is spawned. You must manually copy the data from the old one to the new one using `CopyProperties`.

```cpp
// Inside AMyPlayerState.cpp
void AMyPlayerState::CopyProperties(APlayerState* PlayerState)
{
    Super::CopyProperties(PlayerState);

    if (AMyPlayerState* NewPS = Cast<AMyPlayerState>(PlayerState))
    {
        // Copy custom data across the map transition
        NewPS->TeamID = this->TeamID;
        NewPS->TotalMatchScore = this->TotalMatchScore;
        NewPS->CustomSkinTag = this->CustomSkinTag;
    }
}
```

## 4. Persisting Custom Actors
If you have a special Actor in the world that must survive the travel, the GameMode must explicitly whitelist it using `GetSeamlessTravelActorList`.

```cpp
void AMyGameMode::GetSeamlessTravelActorList(bool bToTransition, TArray<AActor*>& ActorList)
{
    Super::GetSeamlessTravelActorList(bToTransition, ActorList);

    // Keep my custom manager alive
    if (MyPersistentManager)
    {
        ActorList.Add(MyPersistentManager);
    }
}
```

## 5. Impact on Safety
- **Memory Corruption**: If an agent copies pointers (`TObjectPtr<AActor>`) inside `CopyProperties`, those pointers will reference destroyed memory in the new map, crashing the engine. ONLY copy pure data (ints, structs, FNames).
- **Match Continuity**: Failing to implement `CopyProperties` correctly results in players losing their chosen loadouts or team assignments when the match actually starts.

## Verification Checklist
- [ ] `bUseSeamlessTravel` is set to true in the `GameMode`.
- [ ] `CopyProperties` is overridden in `APlayerState` to copy custom variables.
- [ ] `Super::CopyProperties(PlayerState)` is called within the override.
- [ ] NO pointers to Level-specific Actors are copied during travel.
