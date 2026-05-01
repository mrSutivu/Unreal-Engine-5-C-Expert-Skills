---
name: async-task-graph
description: Rules for using AsyncTask and UE::Tasks to offload heavy computations. Preventing Game Thread hitches and Race Conditions.
---

# Async Task Graph (UE5 Expert)

## Activation / When to Use
- Mandatory when generating procedural meshes, executing complex A* pathfinding, or parsing large JSON files.
- Trigger when the Game Thread FPS drops due to `Tick` operations.

## 1. The Game Thread Mandate
Unreal Engine is fundamentally single-threaded for its Object model.
- You CANNOT spawn `AActor`s on a background thread.
- You CANNOT create `UObject`s on a background thread.
- You CANNOT safely read `UPROPERTY`s if the Game Thread might change them simultaneously.

## 2. Using UE::Tasks (Modern UE5)
The new `UE::Tasks` system handles dependencies elegantly.

```cpp
#include "Tasks/Task.h"

void Calculate()
{
    // Launch background work
    UE::Tasks::FTask<int32> CalcTask = UE::Tasks::Launch(TEXT("HeavyCalc"), [] { return 42; });

    // Continuation
    UE::Tasks::Launch(TEXT("PrintResult"), [CalcTask] {
        int32 Result = CalcTask.GetResult();
    }, CalcTask); // Waits for CalcTask
}
```

## 3. Using AsyncTask (Simple/Legacy)
For simple "Go to background, come back" logic.

```cpp
#include "Async/Async.h"

AsyncTask(ENamedThreads::AnyBackgroundThreadNormalTask, []() {
    // Math...
    AsyncTask(ENamedThreads::GameThread, []() {
        // Apply to UI/Actor...
    });
});
```

## 4. Impact on Safety
- **Dangling Pointers**: If you pass `this` (a raw pointer) into an async lambda, and the Actor is destroyed before the background thread finishes, accessing `this->Health` will crash the engine.
- **The Fix**: ALWAYS capture `TWeakObjectPtr<AActor>` or `TWeakObjectPtr<UObject>` and check `.IsValid()` before executing logic inside the lambda.

## Common Mistakes (BAD vs GOOD)

**BAD (Capturing Raw Pointers)**:
```cpp
AsyncTask(ENamedThreads::AnyBackgroundThreadNormalTask, [this]() {
    this->DoSomething(); // CRASH: Actor might be destroyed!
});
```

**GOOD (Capturing Weak Pointers)**:
```cpp
TWeakObjectPtr<AMyActor> WeakThis(this);
AsyncTask(ENamedThreads::AnyBackgroundThreadNormalTask, [WeakThis]() {
    if (WeakThis.IsValid()) { WeakThis->DoSomething(); } // SAFE
});
```

## Verification Checklist
- [ ] `TWeakObjectPtr` is used for all `UObject` references passed into lambdas.
- [ ] `AActor::SpawnActor` and `NewObject` are strictly called on the `GameThread`.
- [ ] Variables are passed *by value* (`[MyVar]`) to avoid race conditions on references.
