---
name: gas-ability-tasks
description: Creating custom asynchronous C++ UAbilityTask classes for the Gameplay Ability System.
---

# GAS: Custom Ability Tasks (UE5 Expert)

## Activation / When to Use
- Mandatory when an Ability needs to wait for a latent action (e.g., waiting for a specific animation frame, a server response, or an overlap event).
- Trigger when the built-in tasks (`PlayMontageAndWait`, `WaitTargetData`) are insufficient.

## 1. Core Principles
`UAbilityTask` is a specialized `UGameplayTask` that manages async operations safely within the context of a `UGameplayAbility`.
- **Lifespan**: Tied to the Ability. If the Ability ends or is cancelled, the Task is automatically destroyed.
- **Delegates**: Uses Dynamic Multicast Delegates to "return" execution paths to the Blueprint/C++ Ability.

## 2. Implementation Syntax

### Header (.h)
```cpp
#pragma once
#include "Abilities/Tasks/AbilityTask.h"
#include "MyAbilityTask.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnTaskFinishedSignature);

UCLASS()
class MYPROJECT_API UAbilityTask_WaitCustomEvent : public UAbilityTask
{
    GENERATED_BODY()

public:
    // Output execution pins in Blueprint
    UPROPERTY(BlueprintAssignable)
    FOnTaskFinishedSignature OnEventFired;

    // Factory method to spawn the task
    UFUNCTION(BlueprintCallable, Category="Ability|Tasks", meta = (HidePin = "OwningAbility", DefaultToSelf = "OwningAbility", BlueprintInternalUseOnly = "TRUE"))
    static UAbilityTask_WaitCustomEvent* CreateWaitCustomEvent(UGameplayAbility* OwningAbility, FName EventName);

    virtual void Activate() override;
    virtual void OnDestroy(bool bInOwnerFinished) override;

protected:
    FName TargetEventName;
    void HandleEvent();
};
```

### Source (.cpp)
```cpp
UAbilityTask_WaitCustomEvent* UAbilityTask_WaitCustomEvent::CreateWaitCustomEvent(UGameplayAbility* OwningAbility, FName EventName)
{
    UAbilityTask_WaitCustomEvent* MyObj = NewAbilityTask<UAbilityTask_WaitCustomEvent>(OwningAbility);
    MyObj->TargetEventName = EventName;
    return MyObj;
}

void UAbilityTask_WaitCustomEvent::Activate()
{
    Super::Activate();
    // Bind to your external event system here
    // e.g., MySystem->OnEvent.AddDynamic(this, &UAbilityTask_WaitCustomEvent::HandleEvent);
}

void UAbilityTask_WaitCustomEvent::HandleEvent()
{
    if (ShouldBroadcastAbilityTaskDelegates())
    {
        OnEventFired.Broadcast();
    }
    // Task complete, kill it
    EndTask();
}

void UAbilityTask_WaitCustomEvent::OnDestroy(bool bInOwnerFinished)
{
    // CRITICAL: Cleanup delegate bindings here
    // MySystem->OnEvent.RemoveDynamic(...)
    Super::OnDestroy(bInOwnerFinished);
}
```

## 3. Impact on Safety
- **Memory Leaks**: `OnDestroy` is guaranteed to be called when the owning Ability terminates. Failing to unbind delegates here will cause the task to live forever as a Zombie.
- **Broadcast Guard**: Using `ShouldBroadcastAbilityTaskDelegates()` prevents the task from firing its execution pin if the ability was cancelled a millisecond prior.

## Common Mistakes (BAD vs GOOD)

**BAD (Missing BlueprintInternalUseOnly)**:
```cpp
UFUNCTION(BlueprintCallable)
static UTask* CreateTask(); // Designer places this node, but it lacks the async execution pins!
```

**GOOD (Correct Meta Tags)**:
```cpp
UFUNCTION(BlueprintCallable, meta=(BlueprintInternalUseOnly = "TRUE")) 
// The engine auto-generates the specialized Async Node UI.
```

## Verification Checklist
- [ ] Factory function uses `NewAbilityTask<T>`.
- [ ] `Activate()` is overridden to start the asynchronous work.
- [ ] `OnDestroy()` is overridden to clean up bindings and pointers.
- [ ] `ShouldBroadcastAbilityTaskDelegates()` wraps all delegate broadcasts.
- [ ] Factory function has `BlueprintInternalUseOnly = "TRUE"`.
