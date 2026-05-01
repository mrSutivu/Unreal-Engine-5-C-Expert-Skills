---
name: bt-service-decorator
description: Creating C++ UBTService (Tick Nodes) and UBTDecorator (Condition Nodes) for Behavior Trees.
---

# Behavior Tree Services & Decorators (UE5 Expert)

## Activation / When to Use
- **Service**: Mandatory when you need a node that runs in the background to update Blackboard variables (e.g., updating distance to player every 0.5s).
- **Decorator**: Mandatory when you need conditional checks to allow/prevent a branch from executing (e.g., "Is Health > 50?").

## 1. UBTService (The Updater)
Services attach to composite nodes (Selectors/Sequences) and "Tick" at a specified interval as long as the branch is active.

### Implementation
```cpp
#pragma once
#include "BehaviorTree/BTService.h"
#include "MyBTService_UpdateDistance.generated.h"

UCLASS()
class MYPROJECT_API UMyBTService_UpdateDistance : public UBTService
{
    GENERATED_BODY()

public:
    UMyBTService_UpdateDistance();

protected:
    virtual void TickNode(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, float DeltaSeconds) override;

    UPROPERTY(EditAnywhere, Category = "Blackboard")
    struct FBlackboardKeySelector DistanceKey;
};
```

```cpp
UMyBTService_UpdateDistance::UMyBTService_UpdateDistance()
{
    NodeName = "Update Distance to Target";
    Interval = 0.5f; // Ticks every 0.5 seconds, not every frame (Optimized)
    RandomDeviation = 0.1f;
}

void UMyBTService_UpdateDistance::TickNode(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, float DeltaSeconds)
{
    Super::TickNode(OwnerComp, NodeMemory, DeltaSeconds);
    // Logic to calculate distance and set blackboard value
}
```

## 2. UBTDecorator (The Gatekeeper)
Decorators attach to any node and evaluate a boolean condition.

### Implementation
```cpp
#pragma once
#include "BehaviorTree/BTDecorator.h"
#include "MyBTDecorator_IsHealthy.generated.h"

UCLASS()
class MYPROJECT_API UMyBTDecorator_IsHealthy : public UBTDecorator
{
    GENERATED_BODY()

public:
    UMyBTDecorator_IsHealthy();

protected:
    virtual bool CalculateRawConditionValue(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory) const override;
};
```

```cpp
UMyBTDecorator_IsHealthy::UMyBTDecorator_IsHealthy()
{
    NodeName = "Is Healthy (> 50%)";
}

bool UMyBTDecorator_IsHealthy::CalculateRawConditionValue(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory) const
{
    AAIController* AIController = OwnerComp.GetAIOwner();
    if (AMyCharacter* Pawn = Cast<AMyCharacter>(AIController->GetPawn()))
    {
        return Pawn->GetHealth() > 50.0f;
    }
    return false; // Fail condition if cast fails
}
```

## 3. Impact on Safety
- **Performance**: Setting a Service `Interval` to 0.0f forces it to tick every frame, destroying CPU performance for large crowds. Always use a reasonable interval (0.2s - 1.0s).
- **Const Correctness**: `CalculateRawConditionValue` is `const`. You CANNOT modify actor state inside a Decorator. It is strictly a read-only query.

## Verification Checklist
- [ ] Services use an `Interval` > 0.0f to avoid ticking every frame.
- [ ] Decorators are strictly read-only and override `CalculateRawConditionValue`.
- [ ] Blackboard Key Selectors are exposed as `EditAnywhere` so designers can map them in the BT Editor.
