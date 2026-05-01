---
name: ai-controller-possession
description: Architecture of AAIController, Blackboards, and Behavior Trees. Rules for AI possession and logic execution.
---

# AI Controller & Possession (UE5 Expert)

## Activation / When to Use
- Mandatory when creating non-player characters (NPCs), enemies, or automated agents.
- Trigger when assigning a Brain (AI Controller) to a Body (Pawn/Character).

## 1. Core Principles
The `AAIController` is the AI equivalent of `APlayerController`.
- **Server Only**: AIControllers exist ONLY on the Server. Clients never execute AI logic. Clients only see the replicated movements and variables of the possessed Pawn.
- **Behavior Tree**: The AIController runs the `UBehaviorTree` and holds the memory `UBlackboardComponent`.

## 2. Configuration & Possession

### Setting up the Pawn
To ensure the AI automatically takes over when spawned, configure the Pawn's Auto Possess settings.
```cpp
AMyEnemyCharacter::AMyEnemyCharacter()
{
    // Possess automatically when spawned or placed in world
    AutoPossessAI = EAutoPossessAI::PlacedInWorldOrSpawned;
    AIControllerClass = AMyAIController::StaticClass();
}
```

### The AIController Initialization
Override `OnPossess` to start the brain logic.

```cpp
#include "BehaviorTree/BlackboardComponent.h"
#include "BehaviorTree/BehaviorTree.h"

void AMyAIController::OnPossess(APawn* InPawn)
{
    Super::OnPossess(InPawn);

    if (AMyEnemyCharacter* Enemy = Cast<AMyEnemyCharacter>(InPawn))
    {
        if (UBehaviorTree* BT = Enemy->GetBehaviorTree())
        {
            // Initializes the Blackboard and starts the Tree
            RunBehaviorTree(BT);
        }
    }
}
```

## 3. Accessing the Blackboard
The Blackboard is the AI's memory (Target Enemy, Patrol Locations).
```cpp
void AMyAIController::SetTargetEnemy(AActor* Target)
{
    if (UBlackboardComponent* BB = GetBlackboardComponent())
    {
        BB->SetValueAsObject(FName("TargetActor"), Target);
    }
}
```

## 4. Impact on Safety
- **Client Execution Fails**: Writing AI logic in a client-rpc or expecting clients to evaluate Behavior Trees will fail completely. AI logic is Server Authority only.
- **Detachment**: When an AI dies, you must detach the controller (`UnPossess()`) or destroy the Controller to stop the Behavior Tree from consuming CPU cycles in the background.

## Common Mistakes (BAD vs GOOD)

**BAD (Client-Side AI)**:
```cpp
// Client attempts to move AI
MyAIController->MoveToLocation(Dest); // Ignored by server, breaks pathfinding.
```

**GOOD (Server-Side AI)**:
```text
All Navigation and MoveTo commands execute on the Server. The built-in CharacterMovementComponent replicates the transform to clients smoothly.
```

## Verification Checklist
- [ ] Pawn has `AutoPossessAI` configured correctly.
- [ ] `AIControllerClass` points to your custom `AAIController`.
- [ ] `RunBehaviorTree()` is called inside `OnPossess()`.
- [ ] Blackboard values are manipulated via `GetBlackboardComponent()`.
