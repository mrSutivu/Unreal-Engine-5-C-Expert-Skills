---
name: blackboard-data-access
description: Safe interaction with UBlackboardComponent from C++. Manipulating Keys using FBlackboardKeySelector.
---

# Blackboard Data Access (UE5 Expert)

## Activation / When to Use
- Mandatory when writing C++ Behavior Tree Tasks, Services, or Decorators that need to read/write memory.
- Trigger when the AI Controller needs to pass a target (Actor or Location) to the Behavior Tree.

## 1. Core Principles
The `UBlackboardComponent` is a dictionary of typed variables (Keys). Accessing them requires the exact `FName` of the key, but the safest method is using `FBlackboardKeySelector`.

## 2. Using Key Selectors (In BT Nodes)
This exposes a dropdown in the BT Editor for the designer to pick the key, avoiding hardcoded strings in C++.

### Header
```cpp
UPROPERTY(EditAnywhere, Category = "Blackboard")
FBlackboardKeySelector TargetActorKey;

UPROPERTY(EditAnywhere, Category = "Blackboard")
FBlackboardKeySelector DestinationVectorKey;
```

### Source
Use the `OwnerComp.GetBlackboardComponent()` to read/write.
```cpp
UBlackboardComponent* BB = OwnerComp.GetBlackboardComponent();

// Writing an Object
BB->SetValueAsObject(TargetActorKey.SelectedKeyName, FoundEnemy);

// Reading a Vector
FVector Dest = BB->GetValueAsVector(DestinationVectorKey.SelectedKeyName);

// Clearing a Key
BB->ClearValue(TargetActorKey.SelectedKeyName);
```

## 3. Direct Access (From AI Controller)
If you are pushing data from the Controller into the Blackboard (not inside a BT Node), you must use hardcoded `FNames`.

```cpp
void AMyAIController::OnSensePlayer(AActor* Player)
{
    if (UBlackboardComponent* BB = GetBlackboardComponent())
    {
        // "TargetEnemy" must perfectly match the key name in the Blackboard Asset
        BB->SetValueAsObject(FName("TargetEnemy"), Player);
    }
}
```

## 4. Impact on Safety
- **Type Mismatch**: Calling `SetValueAsObject` on a key that was defined as a `Vector` in the Blackboard Asset will fail silently. Always double-check types.
- **String Typo Safety**: Hardcoded `FName`s are brittle. If a designer renames the key in the Blackboard Asset, the C++ code breaks silently. `FBlackboardKeySelector` protects against this by linking by reference in the BT Editor.

## Common Mistakes (BAD vs GOOD)

**BAD (Hardcoding in BT Nodes)**:
```cpp
BB->SetValueAsObject(FName("CurrentTarget"), Enemy); // BAD: Breaks if designer renames the key.
```

**GOOD (Key Selector)**:
```cpp
BB->SetValueAsObject(TargetKey.SelectedKeyName, Enemy); // SAFE: Configured in Editor.
```

## Verification Checklist
- [ ] `FBlackboardKeySelector` is used for all BT Node memory access.
- [ ] Hardcoded `FNames` are used ONLY when injecting data from outside the Behavior Tree (e.g., from the AIController).
- [ ] Getters/Setters strictly match the Blackboard Key type (e.g., `GetValueAsEnum`, `SetValueAsBool`).
