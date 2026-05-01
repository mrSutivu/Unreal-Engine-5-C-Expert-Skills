---
name: framework-reference-routing
description: Safe methodologies for traversing the Game Framework hierarchy. Prevents null pointer crashes when accessing Controllers, States, and GameModes.
---

# Framework Reference Routing (UE5 Expert)

## Activation / When to Use
- Mandatory when writing logic that needs to access other parts of the Game Framework.
- Trigger when casting between `Pawn`, `Controller`, `PlayerState`, and `GameState`.

## 1. The Relationship Matrix
Understanding who owns who is critical for pointer safety.
- **World** -> Owns `GameMode` and `GameState`.
- **GameMode** -> Knows all `PlayerControllers`.
- **GameState** -> Knows all `PlayerStates` (via `PlayerArray`).
- **PlayerController** -> Owns exactly ONE `PlayerState`. Possesses exactly ONE `Pawn` (or zero).
- **Pawn** -> Holds a reference to its possessing `Controller` and its `PlayerState`.

## 2. Safe Traversal Patterns

### Pawn to PlayerController
```cpp
// INSIDE APawn or ACharacter
if (APlayerController* PC = Cast<APlayerController>(GetController())) {
    // Safe. Will be null for AI or unpossessed pawns.
}
```

### Pawn to PlayerState
```cpp
// Standard UE5 Template Method
if (AMyPlayerState* PS = GetPlayerState<AMyPlayerState>()) {
    // Safe. Use this instead of Cast<T>(GetPlayerState())
}
```

### Controller to Pawn
```cpp
// INSIDE APlayerController
if (AMyCharacter* MyChar = GetPawn<AMyCharacter>()) {
    // Safe. Returns null if in spectator mode or dead.
}
```

### Anywhere to GameState
```cpp
if (UWorld* World = GetWorld()) {
    if (AMyGameState* GS = World->GetGameState<AMyGameState>()) {
        // Safe.
    }
}
```

## 3. Dealing with Initialization Delays (Networking)
On clients, pointers are not all populated instantly at `BeginPlay`.
- **The Problem**: A client's `Pawn` might spawn before its `PlayerState` is fully replicated.
- **The Fix**: Do not assume `GetPlayerState()` is valid in a Client's `BeginPlay`. Wait for `OnRep_PlayerState()` or check periodically.

## 4. Impact on Safety
- **Silent Cast Failures**: Using hard assumptions (no null checks) when routing framework references is the #1 cause of Access Violation crashes in UE5.
- **Server/Client Context**: Knowing the matrix prevents agents from attempting to route from a `Pawn` to a `GameMode` on a Client (which always fails).

## Common Mistakes (BAD vs GOOD)

**BAD (Chaining without checks)**:
```cpp
// Arrow Code of Death
int32 Score = GetController()->GetPlayerState<AMyPlayerState>()->Kills; // CRASH if unpossessed!
```

**GOOD (Guarded Routing)**:
```cpp
if (AController* Ctrl = GetController()) {
    if (AMyPlayerState* PS = Ctrl->GetPlayerState<AMyPlayerState>()) {
        int32 Score = PS->Kills; // 100% Safe
    }
}
```

## Verification Checklist
- [ ] Safe getter templates (`GetPawn<T>`, `GetPlayerState<T>`) are used over manual `Cast<T>`.
- [ ] Every traversal step checks for `nullptr`.
- [ ] Client logic respects initialization delays for replicated framework pointers.
