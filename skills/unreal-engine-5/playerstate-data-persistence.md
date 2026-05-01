---
name: playerstate-data-persistence
description: Utilizing APlayerState for replicated player data. Ensures player identity, score, and ping survive Pawn destruction.
---

# PlayerState Data Persistence (UE5 Expert)

## Activation / When to Use
- Mandatory for storing replicated data specific to a single player (Score, Ping, PlayerName, Team ID).
- Trigger when you need player data to survive the death of their Pawn.

## 1. Core Principles
The `APlayerState` represents the state of a participant in the game.
- **Global Replication**: Created on the Server, replicated to ALL clients. Everyone knows everyone else's PlayerState.
- **Decoupling from Body**: When a player dies (`APawn` destroyed), the `APlayerState` remains intact.
- **Seamless Travel**: By default, `APlayerState` can carry data across level transitions using `CopyProperties`.

## 2. Responsibilities
- Broadcasting stats (Kills, Deaths, Assists) to the GameState's scoreboard.
- Replicating cosmetic data (e.g., Chosen Character Skin, Title).
- Providing peer-to-peer information (Client A reading Client B's data).

## 3. Impact on Safety
- **Late Joiners (JIP)**: When a player joins late, they receive the `GameState` and all current `PlayerState`s automatically. Agents must use `PlayerState` for any logic that JIP players need to see.
- **Pointer Volatility**: The `PlayerState` pointer inside a `Pawn` is not guaranteed to be valid immediately in `BeginPlay` on clients. Use `OnRep_PlayerState` or poll for it.

## Common Mistakes (BAD vs GOOD)

**BAD (Score inside Pawn)**:
```cpp
UCLASS()
class AMyCharacter : public ACharacter {
    UPROPERTY(Replicated)
    int32 Kills; // BAD: When the Character dies, the kill count is deleted!
};
```

**GOOD (Score inside PlayerState)**:
```cpp
UCLASS()
class AMyPlayerState : public APlayerState {
    UPROPERTY(ReplicatedUsing = OnRep_Kills)
    int32 Kills; // SAFE: Survives respawns and replicates to all clients.
};
```

**BAD (Unsafe Casting in BeginPlay)**:
```cpp
void AMyCharacter::BeginPlay() {
    // Might be null on clients if replication is delayed!
    FString Name = GetPlayerState()->GetPlayerName(); 
}
```

**GOOD (Handling Replication Delay)**:
```cpp
void AMyCharacter::OnRep_PlayerState() {
    Super::OnRep_PlayerState();
    if (GetPlayerState()) {
        // Safe to read data now
    }
}
```

## Verification Checklist
- [ ] Persistent player data (Score, Team, Name) is housed in `APlayerState`.
- [ ] Variables are marked `UPROPERTY(Replicated)`.
- [ ] `OnRep_PlayerState` is overridden in the Pawn to handle delayed initialization on clients.
- [ ] `PlayerState` is NOT used for secure inputs or Server RPCs (Use `PlayerController`).
