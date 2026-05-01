---
name: ue5coro-coroutines
description: Implementing C++20 Coroutines in UE5 using the UE5Coro plugin. Writing synchronous-looking asynchronous code.
---

# UE5Coro & Coroutines (UE5 Expert)

## Activation / When to Use
- Mandatory if the project uses C++20 and the `UE5Coro` plugin (or native UE5.3+ coroutine wrappers).
- Trigger when you have complex sequential logic that requires waiting (e.g., "Wait for animation, then wait for network, then spawn").

## 1. Core Principles
Coroutines (`co_await`, `co_return`) allow a function to suspend its execution without blocking the thread. It eliminates "Callback Hell" (nested delegates).

## 2. Implementation Syntax

### Header (.h)
The function must return `UE5Coro::TCoroutine<>` (or `FCoroTask` in some native implementations).
```cpp
#pragma once
#include "UE5Coro.h"

UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()
public:
    UE5Coro::TCoroutine<> RunComplexSequence();
};
```

### Source (.cpp)
```cpp
#include "UE5Coro.h"

UE5Coro::TCoroutine<> AMyActor::RunComplexSequence()
{
    // 1. Suspend execution for 2 seconds (Does NOT block the Game Thread)
    co_await UE5Coro::Latent::Seconds(2.0f);

    // 2. Move to a background thread to do heavy math
    co_await UE5Coro::Async::MoveToBackground();
    float Result = CalculateHeavyMath();

    // 3. Move back to the Game Thread to spawn an actor
    co_await UE5Coro::Async::MoveToGameThread();
    GetWorld()->SpawnActor<AExplosion>(...);
}
```

## 3. Awaiting Delegates
You can `co_await` a delegate, completely removing the need to write separate callback functions.

```cpp
UE5Coro::TCoroutine<> AMyCharacter::AttackAndWait()
{
    PlayAnimMontage(AttackMontage);
    
    // Wait for the specific delegate to broadcast
    co_await UE5Coro::Async::AwaitDelegate(OnMontageEnded);
    
    // Attack finished, proceed
    EquipNextWeapon();
}
```

## 4. Impact on Safety
- **Actor Destruction**: `UE5Coro` safely cancels the coroutine if the `UObject` it is bound to is destroyed while suspended. No manual cleanup needed.
- **Readability**: Code is top-to-bottom, vastly reducing cognitive load ("Oversight Demand") compared to managing 5 separate callback functions.

## Common Mistakes (BAD vs GOOD)

**BAD (Delegate Hell)**:
```cpp
void Start() { PlayAnim(); OnEnd.AddDynamic(this, &A::Step2); }
void Step2() { DownloadData(); OnDownload.AddDynamic(this, &A::Step3); }
void Step3() { ... }
```

**GOOD (Coroutine)**:
```cpp
TCoroutine<> Start() {
    PlayAnim();
    co_await OnEnd;
    DownloadData();
    co_await OnDownload;
}
```

## Verification Checklist
- [ ] Function returns `TCoroutine<>` or engine equivalent.
- [ ] `co_await` is used for delays, delegate waits, and thread swapping.
- [ ] Project is configured to compile with C++20.
