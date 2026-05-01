---
name: gamestate-global-replication
description: Usage of AGameStateBase for global replicated data. Separates shared match state from isolated PlayerState data.
---

# GameState Global Replication (UE5 Expert)

## Activation / When to Use
- Mandatory when clients need to know the state of the game (e.g., Match Time, Team Scores, Objective Progress).
- Trigger when data must be available to everyone in the session.

## 1. Core Principles
`AGameStateBase` is the public face of the GameMode. 
- **Replicated**: Created on the server and automatically replicated to ALL clients.
- **Global Availability**: Easily accessible from anywhere via `GetWorld()->GetGameState<T>()`.
- **Player Array**: Automatically maintains a replicated array of all `APlayerState` objects currently in the game (`PlayerArray`).

## 2. Responsibilities
- Keeping track of global variables (`TeamScore`, `RemainingTime`).
- Firing global Multicast RPCs or RepNotifies for match events (e.g., `OnRep_MatchStateChanged`).

## 3. Impact on Safety
- **Network Bandwidth**: Do not put fast-changing, per-player data in GameState (use `PlayerState` instead). Overloading GameState causes severe network bottlenecks.
- **Initialization Race Conditions**: Clients receive the GameState *after* their PlayerController connects. Agents must account for the GameState being `nullptr` during early `BeginPlay` on clients.

## Common Mistakes (BAD vs GOOD)

**BAD (Data in GameMode)**:
```cpp
// Client tries to read score but GameMode doesn't exist on Client.
int32 Score = GetWorld()->GetAuthGameMode<AMyGameMode>()->TeamScore; // Nullptr!
```

**GOOD (Data in GameState)**:
```cpp
if (AMyGameState* GS = GetWorld()->GetGameState<AMyGameState>()) {
    int32 Score = GS->TeamScore; // SAFE. Replicated to client.
}
```

**BAD (Per-Player Data in GameState)**:
```cpp
UPROPERTY(Replicated)
TArray<FString> EveryPlayersLoadout; // Huge replication cost on one actor.
```

**GOOD (Per-Player Data Distributed)**:
```text
Move player loadouts to individual `APlayerState` objects.
```

## Verification Checklist
- [ ] Data needed by clients about the general match is stored here.
- [ ] Variables are marked `UPROPERTY(Replicated)` and registered in `GetLifetimeReplicatedProps`.
- [ ] Client access to `GetGameState()` includes a `nullptr` check (due to network replication delay).
- [ ] `PlayerArray` is used to iterate over players instead of `GetAllActorsOfClass`.
