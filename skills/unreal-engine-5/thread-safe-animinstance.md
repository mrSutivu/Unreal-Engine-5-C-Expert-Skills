---
name: thread-safe-animinstance
description: Utilizing NativeThreadSafeUpdateAnimation in UE5. Ensures animation logic runs off the Game Thread to maximize performance.
---

# Thread-Safe AnimInstance (UE5 Expert)

## Activation / When to Use
- Mandatory when building C++ AnimInstances (`UAnimInstance` overrides).
- Trigger when preparing an animation graph for high-performance multi-threading (Character crowds, high framerate).

## 1. The UE5 Thread-Safe Mandate
In UE4, developers used `NativeUpdateAnimation` to copy variables from the Pawn to the AnimInstance. This runs on the Game Thread, creating massive CPU bottlenecks for crowds.
In UE5, `NativeThreadSafeUpdateAnimation` runs asynchronously on Worker Threads.

## 2. Implementation Syntax

### Header (.h)
```cpp
#pragma once
#include "Animation/AnimInstance.h"
#include "MyAnimInstance.generated.h"

UCLASS()
class MYPROJECT_API UMyAnimInstance : public UAnimInstance
{
    GENERATED_BODY()

public:
    virtual void NativeInitializeAnimation() override;
    
    // THE UE5 STANDARD (Runs on Worker Thread)
    virtual void NativeThreadSafeUpdateAnimation(float DeltaSeconds) override;

protected:
    UPROPERTY(BlueprintReadOnly, Category = "Movement")
    float Speed;

    UPROPERTY(BlueprintReadOnly, Category = "Movement")
    bool bIsFalling;

    // Cached pointer to the movement component
    UPROPERTY()
    class UCharacterMovementComponent* MovementComponent;
};
```

### Source (.cpp)
You CANNOT call typical `UObject` functions or read complex actor states directly in a Thread-Safe function because the Game Thread might be modifying them simultaneously (Race Condition).
- **Solution**: Cache components in `Initialize`, and use the `PropertyAccess` system (or direct atomic reads if safe) in `Update`.

```cpp
void UMyAnimInstance::NativeInitializeAnimation()
{
    Super::NativeInitializeAnimation();

    if (APawn* Owner = TryGetPawnOwner())
    {
        MovementComponent = Owner->FindComponentByClass<UCharacterMovementComponent>();
    }
}

void UMyAnimInstance::NativeThreadSafeUpdateAnimation(float DeltaSeconds)
{
    Super::NativeThreadSafeUpdateAnimation(DeltaSeconds);

    // Ensure pointers are valid (accessed atomically)
    if (MovementComponent)
    {
        // Read velocities and states safely
        Speed = MovementComponent->Velocity.Size2D();
        bIsFalling = MovementComponent->IsFalling();
    }
}
```

## 3. Blueprint Thread-Safe Access
To access data safely inside the AnimGraph, developers should use "Property Access" nodes (the nodes with a little stopwatch icon). As an agent, ensure C++ exposes variables as `BlueprintReadOnly` so they can be accessed via fast paths.

## 4. Impact on Safety
- **Race Conditions**: Calling `TryGetPawnOwner()->GetActorRotation()` inside `NativeThreadSafeUpdateAnimation` can cause a memory crash if the Actor is destroyed or modified by the Game Thread simultaneously. Always cache safe pointers or use `GetOwningComponent()`.
- **Performance**: Thread-safe animation updates allow hundreds of characters to animate with near-zero impact on the main Game Thread.

## Common Mistakes (BAD vs GOOD)

**BAD (UE4 Legacy/Unsafe Math)**:
```cpp
void UMyAnimInstance::NativeUpdateAnimation(float DeltaSeconds) {
    Speed = TryGetPawnOwner()->GetVelocity().Size(); // Blocks Game Thread!
}
```

**GOOD (UE5 Thread-Safe)**:
```cpp
void UMyAnimInstance::NativeThreadSafeUpdateAnimation(float DeltaSeconds) {
    if (MovementComponent) { Speed = MovementComponent->Velocity.Size2D(); } // Runs in background
}
```

## Verification Checklist
- [ ] `NativeThreadSafeUpdateAnimation` is used instead of `NativeUpdateAnimation`.
- [ ] `TryGetPawnOwner()` is only called in `NativeInitializeAnimation` to cache pointers.
- [ ] No complex Engine functions (like Traces or Actor Spawning) are called in the Thread-Safe update.
