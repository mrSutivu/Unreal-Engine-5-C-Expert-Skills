---
name: gas-ugameplayability
description: Implementation of UGameplayAbility. Covering Instancing Policies, CommitAbility, and EndAbility lifecycles.
---

# GAS: UGameplayAbility (UE5 Expert)

## Activation / When to Use
- Mandatory when defining actions an Actor can perform (e.g., Attack, Sprint, Cast Spell) within the Gameplay Ability System.
- Trigger when you need networked, state-aware actions with cooldowns and costs.

## 1. Instancing Policies
An ability dictates how it is spawned into memory via its `InstancingPolicy`.

- **NonInstanced**: Cheapest. The ability logic runs on the Class Default Object (CDO). You CANNOT have local variables in the class. Best for simple attacks or passive buffs.
- **InstancedPerActor**: Created once per Avatar and reused. Allows state variables (e.g., combo counters). Best for primary player abilities.
- **InstancedPerExecution**: Spawns a new UObject every time the ability is activated. Expensive. Rarely used unless the ability has heavily threaded asynchronous tasks.

## 2. Implementation Syntax

### Header (.h)
```cpp
UCLASS()
class MYPROJECT_API UMyGameplayAbility : public UGameplayAbility
{
    GENERATED_BODY()

public:
    UMyGameplayAbility();

    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData) override;
    virtual void EndAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, bool bReplicateEndAbility, bool bWasCancelled) override;
};
```

### Source (.cpp)
```cpp
UMyGameplayAbility::UMyGameplayAbility()
{
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    
    // Defines what tags are granted while active, or what tags block activation
    // e.g., Blocked by "State.Dead"
}

void UMyGameplayAbility::ActivateAbility(...)
{
    // MUST call CommitAbility to apply Costs and Cooldowns
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // Ability Logic (Play Montage, Wait for Input, etc.)
    // ...

    // ALWAYS call EndAbility when finished
    EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
}
```

## 3. Tasks & Asynchronous Execution
Abilities use `UAbilityTask` to handle asynchronous actions (e.g., waiting for an animation to finish or a player to press a key).

```cpp
// In Blueprint or C++, spawning a task
UAbilityTask_PlayMontageAndWait* Task = UAbilityTask_PlayMontageAndWait::CreatePlayMontageAndWaitProxy(this, NAME_None, AttackMontage);
Task->OnCompleted.AddDynamic(this, &UMyGameplayAbility::OnMontageFinished);
Task->ReadyForActivation(); // CRITICAL: Never forget to call this!
```

## 4. Impact on Safety
- **Commit Constraints**: Failing to call `CommitAbility` means the ability ignores cooldowns and mana costs, allowing players to spam infinite attacks.
- **EndAbility Lock**: Failing to call `EndAbility` leaves the ability "Active" forever. If it blocks other abilities via tags, the player becomes permanently stuck (Softlock).

## Common Mistakes (BAD vs GOOD)

**BAD (Variables in NonInstanced)**:
```cpp
UCLASS()
class UMyAbility : public UGameplayAbility {
    // InstancingPolicy = NonInstanced;
    int32 HitCount; // FATAL: Modifies the CDO. Affects all players simultaneously!
};
```

**GOOD (Stateful Instancing)**:
```text
Set InstancingPolicy to `InstancedPerActor` if you need state variables.
```

**BAD (Forgetting ReadyForActivation)**:
```text
Creating an AbilityTask but forgetting `Task->ReadyForActivation()`. The task will silently fail to execute.
```

## Verification Checklist
- [ ] `InstancingPolicy` is explicitly chosen in the constructor.
- [ ] `CommitAbility` is checked before executing payload logic.
- [ ] Every possible execution path leads to `EndAbility`.
- [ ] `ReadyForActivation()` is called on all instantiated `UAbilityTasks`.
