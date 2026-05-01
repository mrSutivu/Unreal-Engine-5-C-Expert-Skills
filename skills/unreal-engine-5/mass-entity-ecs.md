---
name: mass-entity-ecs
description: Architecture of the Mass Entity Component System (ECS). Simulating 100,000+ entities with FMassFragment and UMassProcessor.
---

# Mass Entity ECS (UE5 Expert)

## Activation / When to Use
- Mandatory when simulating tens of thousands of entities (Crowds, Traffic, Boids, Flocks).
- Trigger when traditional `AActor` + `UCharacterMovementComponent` drops the framerate below 60fps at ~100 units.

## 1. Core Principles (Data-Oriented Design)
Mass Entity abandons the Object-Oriented `AActor`. 
- **Entity**: Just an Integer ID.
- **Fragment**: A raw struct containing ONLY data (e.g., `FVector Location`).
- **Processor**: A C++ class that contains ONLY logic. It iterates over arrays of Fragments.

*Result*: Perfect CPU Cache Locality. Unmatched performance.

## 2. Implementation Syntax

### The Fragment (Data)
```cpp
#pragma once
#include "MassEntityTypes.h"
#include "MyMassFragment.generated.h"

USTRUCT()
struct FMassVelocityFragment : public FMassFragment
{
    GENERATED_BODY()
    FVector Velocity = FVector::ZeroVector;
};
```

### The Processor (Logic)
Iterates over all entities that have a Velocity and a Transform fragment.

```cpp
#pragma once
#include "MassProcessor.h"
#include "MyMassProcessor.generated.h"

UCLASS()
class UMyMovementProcessor : public UMassProcessor
{
    GENERATED_BODY()
public:
    UMyMovementProcessor();

protected:
    virtual void ConfigureQueries() override;
    virtual void Execute(UMassEntitySubsystem& EntitySubsystem, FMassExecutionContext& Context) override;

    FMassEntityQuery EntityQuery;
};
```

### The Execution (Source)
```cpp
#include "MassCommonFragments.h"

UMyMovementProcessor::UMyMovementProcessor()
{
    // Run on every frame
    ExecutionFlags = (int32)EProcessorExecutionFlags::All;
}

void UMyMovementProcessor::ConfigureQueries()
{
    // Define what data this processor needs
    EntityQuery.AddRequirement<FTransformFragment>(EMassFragmentAccess::ReadWrite);
    EntityQuery.AddRequirement<FMassVelocityFragment>(EMassFragmentAccess::ReadOnly);
}

void UMyMovementProcessor::Execute(UMassEntitySubsystem& EntitySubsystem, FMassExecutionContext& Context)
{
    // Execute the query
    EntityQuery.ForEachEntityChunk(EntitySubsystem, Context, [](FMassExecutionContext& Context)
    {
        // Get the arrays of data for this chunk (Extremely fast)
        TArrayView<FTransformFragment> Transforms = Context.GetMutableFragmentView<FTransformFragment>();
        TArrayView<const FMassVelocityFragment> Velocities = Context.GetFragmentView<FMassVelocityFragment>();

        float DeltaTime = Context.GetDeltaTimeSeconds();

        // Loop through all entities in the chunk
        for (int32 EntityIndex = 0; EntityIndex < Context.GetNumEntities(); ++EntityIndex)
        {
            FVector Translation = Velocities[EntityIndex].Velocity * DeltaTime;
            Transforms[EntityIndex].GetMutableTransform().AddToTranslation(Translation);
        }
    });
}
```

## 3. Impact on Safety
- **No Virtual Calls**: There are no `Tick()` or `BeginPlay()` overrides for individual entities. The Processor executes everything in a flat loop, preventing call-stack bloat.
- **Access Violations**: `EMassFragmentAccess::ReadOnly` vs `ReadWrite` MUST be respected. Mass schedules processors asynchronously based on these access flags. Lying about access will cause data races and crashes.

## Verification Checklist
- [ ] Fragments inherit from `FMassFragment` and contain ONLY data (no functions).
- [ ] Processors inherit from `UMassProcessor`.
- [ ] Queries use `EMassFragmentAccess` correctly to enable parallel execution safely.
- [ ] Iteration is done via `ForEachEntityChunk` and `TArrayView`.
