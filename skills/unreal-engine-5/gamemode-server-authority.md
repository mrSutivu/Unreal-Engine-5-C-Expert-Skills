---
name: gamemode-server-authority
description: Architectural rules for AGameMode and AGameModeBase. Enforces server-only authority, match states, and safe client access prevention.
---

# GameMode Server Authority (UE5 Expert)

## Activation / When to Use
- Mandatory when defining game rules, spawning logic, and match states.
- Trigger when designing code that must ONLY run on the server.

## 1. Core Principles
The `AGameModeBase` (and its child `AGameMode`) dictates the rules of the game.
- **Server Only**: It is NEVER replicated to clients. It exists ONLY on the Authority (Server) and Standalone games.
- **Instantiation**: Created exactly once per World/Level.

## 2. Responsibilities
- Class Defaults: Defines the default `Pawn`, `PlayerController`, `PlayerState`, and `GameState` classes.
- Spawning: Decides WHERE and WHEN a player spawns (`ChoosePlayerStart`, `RestartPlayer`).
- Match Flow (AGameMode only): Handles `WaitingToStart`, `InProgress`, `MatchHasEnded` states.

## 3. Impact on Safety
- **Client Null Pointers**: Calling `GetWorld()->GetAuthGameMode()` on a Client returns `nullptr`. If the agent fails to check this, the client will crash.
- **Cheat Prevention**: Because clients cannot access the GameMode, placing win-conditions or inventory validation here guarantees authoritative security.

## Common Mistakes (BAD vs GOOD)

**BAD (Client Crash)**:
```cpp
void AMyCharacter::BeginPlay() {
    // CRASH ON CLIENT: GetAuthGameMode() returns nullptr on clients!
    int32 MaxScore = GetWorld()->GetAuthGameMode<AMyGameMode>()->MaxScore; 
}
```

**GOOD (Server-Side Guard)**:
```cpp
void AMyCharacter::BeginPlay() {
    if (HasAuthority()) {
        if (AMyGameMode* GM = GetWorld()->GetAuthGameMode<AMyGameMode>()) {
            int32 MaxScore = GM->MaxScore; // SAFE
        }
    }
}
```

**BAD (Replicating Data via GameMode)**:
```cpp
// In AMyGameMode.h
UPROPERTY(Replicated) // FATAL ERROR: GameMode is never replicated.
int32 TeamAScore;
```

**GOOD (Using GameState for Replication)**:
```text
Move `TeamAScore` to `AGameStateBase` so clients can read it.
```

## Verification Checklist
- [ ] Logic requiring client knowledge is moved to `GameState`.
- [ ] `GetAuthGameMode()` is guarded by `HasAuthority()` or `nullptr` checks.
- [ ] GameMode handles `Login`, `Logout`, and `SpawnDefaultPawnAtTransform`.
