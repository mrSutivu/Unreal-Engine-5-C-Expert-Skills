---
name: multicast-dynamic-delegates
description: Expert implementation of DECLARE_DYNAMIC_MULTICAST_DELEGATE. Enables C++ systems to broadcast events to multiple Blueprint listeners for UI and gameplay triggers.
---

# Multicast Dynamic Delegates (UE5 Expert)

## Activation / When to Use
- Mandatory when a C++ event must be observable by multiple Blueprints (e.g., UI updates, state changes).
- Trigger when you need decoupled "Fire and Forget" event broadcasting.

## 1. Core Principles
- **Dynamic**: Can be serialized (saved) and bound within the Blueprint Graph.
- **Multicast**: Allows multiple functions (listeners) to bind to the same event.
- **Decoupling**: The sender does not know or care who is listening.

## 2. Implementation Syntax

### Declaration
Must be declared OUTSIDE the class definition. Max 8 parameters.
```cpp
// Delegate with 1 parameter
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnHealthChangedSignature, float, NewHealth);
```

### Class Member
Must be a `BlueprintAssignable` UPROPERTY.
```cpp
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    UPROPERTY(BlueprintAssignable, Category = "Events")
    FOnHealthChangedSignature OnHealthChanged;
};
```

### Broadcasting (Firing)
Always check if bound (though Multicast is safe to broadcast empty).
```cpp
void AMyCharacter::TakeDamage(float Amount)
{
    Health -= Amount;
    OnHealthChanged.Broadcast(Health);
}
```

## 3. Impact on Safety
- **Blueprint Isolation**: Prevents C++ from needing hard references to UI classes.
- **Memory Safety**: `Dynamic` delegates use weak references internally. If a Blueprint object is destroyed, it is safely unbound automatically.

## Common Mistakes (BAD vs GOOD)

**BAD (Hard Reference to UI)**:
```cpp
void AMyCharacter::TakeDamage() {
    MyUIWidget->UpdateHealthBar(); // Tightly coupled, breaks if UI changes
}
```

**GOOD (Delegate Broadcast)**:
```cpp
void AMyCharacter::TakeDamage() {
    OnHealthChanged.Broadcast(Health); // Decoupled. UI listens if it wants to.
}
```

## Verification Checklist
- [ ] Delegate names end with `Signature` (convention).
- [ ] UPROPERTY is marked `BlueprintAssignable`.
- [ ] Delegate is declared outside the class.
- [ ] No hard references to listening classes exist in the broadcaster.
