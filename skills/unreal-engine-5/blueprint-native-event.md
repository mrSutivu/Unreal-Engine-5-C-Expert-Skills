---
name: blueprint-native-event
description: Implementation of BlueprintNativeEvent for functions requiring a C++ default behavior that can be overridden by Blueprint designers.
---

# BlueprintNativeEvent Implementation (UE5 Expert)

## Activation / When to Use
- Mandatory when a function needs a baseline C++ implementation BUT designers must be able to override or extend it in Blueprint.
- Essential for core gameplay logic like `Interact`, `TakeDamage`, or `OnDeath`.

## 1. Core Principles
- **Dual Identity**: Generates a C++ thunk that calls the Blueprint graph if an override exists. If not, it calls the native `_Implementation`.
- **Extensibility**: Designers can choose to replace the C++ logic entirely or append to it by calling "Add call to parent function".

## 2. Implementation Syntax

### Declaration (.h)
Do NOT declare the `_Implementation` function manually in modern UE5 (UHT does it), just define it in the `.cpp`.
```cpp
UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category = "Interaction")
void Interact(AActor* Instigator);
```

### Definition (.cpp)
You MUST append `_Implementation` to the function name.
```cpp
void AMyActor::Interact_Implementation(AActor* Instigator)
{
    // Default C++ behavior
    UE_LOGFMT(LogTemp, Log, "Base C++ Interaction logic executed.");
}
```

### Calling the Event
Call the original name, NOT the `_Implementation`.
```cpp
MyActor->Interact(PlayerPawn);
```

## 3. Impact on Safety
- **Fallback Security**: Guarantees that if a designer creates a child Blueprint and forgets to implement the event, the game will not break (falls back to C++).
- **Oversight**: Allows architects to enforce core security checks in C++ while giving visual flair control to BP.

## Common Mistakes (BAD vs GOOD)

**BAD (Calling Implementation directly)**:
```cpp
MyActor->Interact_Implementation(Pawn); // Bypasses Blueprint entirely!
```

**GOOD (Correct call)**:
```cpp
MyActor->Interact(Pawn); // Routes through BP first, then C++ if needed.
```

**BAD (Manual Header Declaration)**:
```cpp
// Header
void Interact(AActor* Instigator);
virtual void Interact_Implementation(AActor* Instigator); // REDUNDANT in modern UHT
```

## Verification Checklist
- [ ] `BlueprintNativeEvent` is paired with `BlueprintCallable` if C++ needs to call it.
- [ ] Native logic is written exclusively inside the `_Implementation` function.
- [ ] Original function name is used when invoking the event.
