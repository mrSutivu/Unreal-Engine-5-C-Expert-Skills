---
name: interface-implementation
description: Creating modular C++ interfaces (U/I prefix) to communicate between diverse Actor types without hard-casting.
---

# Interface Implementation (UE5 Expert)

## Activation / When to Use
- Mandatory when disparate classes need to share functionality (e.g., `Interact()`, `TakeDamage()`).
- Use to avoid heavy inheritance trees or expensive `Cast<T>` operations.

## 1. Core Principles
- **Two Classes**: Unreal interfaces require a `U` class (for reflection) and an `I` class (for native C++ implementation).
- **Decoupling**: Allows an agent to call a function on an Actor without knowing what the Actor actually is.

## 2. Implementation Syntax

### Declaration (.h)
```cpp
#pragma once
#include "CoreMinimal.h"
#include "UObject/Interface.h"
#include "InteractableInterface.generated.h"

// U-Class for Reflection (Do not modify)
UINTERFACE(MinimalAPI, Blueprintable)
class UInteractableInterface : public UInterface { GENERATED_BODY() };

// I-Class for Implementation
class IInteractableInterface
{
    GENERATED_BODY()

public:
    // BlueprintNativeEvent is highly recommended for Interfaces
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category = "Interaction")
    void Interact(AActor* Instigator);
};
```

### Implementing in an Actor (.h)
```cpp
class AMyDoor : public AActor, public IInteractableInterface
{
    GENERATED_BODY()
public:
    virtual void Interact_Implementation(AActor* Instigator) override;
};
```

## 3. Calling Interface Functions
CRITICAL: You must use the `Execute_` static function to call `BlueprintNativeEvent` interfaces safely.

```cpp
void APlayer::TryInteract(AActor* Target)
{
    // Check if target implements the interface
    if (Target && Target->Implements<UInteractableInterface>())
    {
        // SAFE CALL: Routes through BP or C++ correctly
        IInteractableInterface::Execute_Interact(Target, this);
    }
}
```

## 4. Impact on Safety
- **Cast Elimination**: Hard casts (`Cast<AMyDoor>`) create hard references, bloating memory. Interfaces keep systems modular and lightweight.
- **Graceful Failure**: Checking `Implements<T>()` ensures the system won't crash if the target doesn't support the action.

## Common Mistakes (BAD vs GOOD)

**BAD (Direct Interface Cast for BP events)**:
```cpp
IInteractableInterface* IntObj = Cast<IInteractableInterface>(Target);
if (IntObj) { IntObj->Interact(this); } // FAILS if implemented only in Blueprint!
```

**GOOD (Execute_ Syntax)**:
```cpp
if (Target->Implements<UInteractableInterface>()) {
    IInteractableInterface::Execute_Interact(Target, this); // Works for BP and C++
}
```

## Verification Checklist
- [ ] Interface file contains both `U` and `I` classes.
- [ ] `Blueprintable` added to `UINTERFACE` macro if Blueprints need to implement it.
- [ ] `Execute_FunctionName` is used to invoke the interface method.
- [ ] `Implements<UInterfaceClass>()` is used to verify support before calling.
