---
name: anim-notifies-cpp
description: Creating C++ UAnimNotify and UAnimNotifyState classes. Triggering sounds, VFX, and hitboxes at specific animation frames.
---

# AnimNotifies in C++ (UE5 Expert)

## Activation / When to Use
- Mandatory when you need an event to fire at an EXACT frame of an animation (e.g., Footstep sound, Sword swing damage window).
- Trigger when Blueprint AnimNotifies become too numerous, complex, or slow.

## 1. UAnimNotify (One-Shot Event)
Fires a single event at a specific frame.

### Implementation
```cpp
#pragma once
#include "Animation/AnimNotifies/AnimNotify.h"
#include "AnimNotify_PlayFootstep.generated.h"

UCLASS()
class MYPROJECT_API UAnimNotify_PlayFootstep : public UAnimNotify
{
    GENERATED_BODY()

public:
    virtual void Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, const FAnimNotifyEventReference& EventReference) override;
};
```

```cpp
void UAnimNotify_PlayFootstep::Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, const FAnimNotifyEventReference& EventReference)
{
    Super::Notify(MeshComp, Animation, EventReference);

    if (MeshComp && MeshComp->GetOwner())
    {
        // Execute C++ logic (e.g., trace downwards, play sound based on physical surface)
    }
}
```

## 2. UAnimNotifyState (Duration Event)
Has a Begin, Tick, and End. Perfect for Melee Hitboxes (Damage Windows) or Invincibility frames.

### Implementation
```cpp
UCLASS()
class MYPROJECT_API UAnimNotifyState_MeleeWindow : public UAnimNotifyState
{
    GENERATED_BODY()

public:
    virtual void NotifyBegin(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, float TotalDuration, const FAnimNotifyEventReference& EventReference) override;
    
    virtual void NotifyTick(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, float FrameDeltaTime, const FAnimNotifyEventReference& EventReference) override;
    
    virtual void NotifyEnd(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, const FAnimNotifyEventReference& EventReference) override;
};
```

## 3. Impact on Safety & Performance
- **CDO Isolation**: AnimNotifies act like singletons. The same `UAnimNotify` object instance is shared across ALL characters playing that animation. You **CANNOT** store state variables (like `bool bHasHit`) inside the AnimNotify class. It will affect all characters simultaneously.
- **Routing**: Because Notifies are stateless, they should extract the `GetOwner()` from the `MeshComp` and call a function on the Actor to handle the actual stateful logic.

## Common Mistakes (BAD vs GOOD)

**BAD (Stateful Notify)**:
```cpp
class UAnimNotifyState_Melee : public UAnimNotifyState {
    TArray<AActor*> HitActors; // FATAL: Shared between all characters!
};
```

**GOOD (Stateless Routing)**:
```cpp
void UAnimNotifyState_Melee::NotifyTick(...) {
    if (AMyCharacter* Char = Cast<AMyCharacter>(MeshComp->GetOwner())) {
        Char->PerformMeleeTrace(); // Safe: The character tracks who it has hit.
    }
}
```

## Verification Checklist
- [ ] Subclass is `UAnimNotify` (instant) or `UAnimNotifyState` (duration).
- [ ] No member variables (state) are stored in the Notify class.
- [ ] Logic routes to `MeshComp->GetOwner()` to execute stateful behaviors.
