---
name: performance-tick-event-driven
description: Eradicating the Tick function. Moving to an Event-Driven architecture using Delegates and Timers to optimize CPU overhead.
---

# Tick Eradication & Event-Driven Logic (UE5 Expert)

## Activation / When to Use
- Mandatory for ALL new Actor and Component creations.
- Trigger when you see a `Tick` function being used to "check" if a state has changed.

## 1. The Core Principle (Zero Tick Policy)
The `Tick` function is the biggest performance killer in Unreal Engine. Calling `Tick` on 1,000 actors every frame just to check `if (Health <= 0)` wastes massive CPU resources.

- **Rule**: If an Actor doesn't absolutely *need* to update a visual position every single frame (like a homing missile), its Tick must be disabled.

## 2. Disabling Tick
Always disable it in the constructor.

```cpp
AMyActor::AMyActor()
{
    // 1. Disable ticking entirely
    PrimaryActorTick.bCanEverTick = false;
    
    // 2. If it MUST tick, never start it on BeginPlay automatically
    PrimaryActorTick.bStartWithTickEnabled = false;
}
```

## 3. Event-Driven Architecture (The Replacement)
Instead of polling state in `Tick`, push state changes using Delegates (Events).

**BAD (Tick Polling)**:
```cpp
void AMyDoor::Tick(float DeltaTime)
{
    // Scanning every frame! VERY SLOW!
    if (FVector::DistSquared(Player->GetActorLocation(), GetActorLocation()) < 10000) { OpenDoor(); }
}
```

**GOOD (Event-Driven Overlap)**:
```cpp
void AMyDoor::BeginPlay()
{
    // Waiting for the physics engine to tell us instantly when it happens. 0 CPU cost.
    TriggerBox->OnComponentBeginOverlap.AddDynamic(this, &AMyDoor::OnPlayerEnter);
}
```

## 4. Timers vs Tick
If you need a continuous update that doesn't rely on physics (e.g., Poison damage over time, or an AI recalculating a path), use `FTimerManager` instead of `Tick`.

```cpp
void AMyActor::ApplyPoison()
{
    // Ticks every 1.0 seconds, instead of 60 times a second.
    GetWorldTimerManager().SetTimer(PoisonTimer, this, &AMyActor::TakePoisonDamage, 1.0f, true);
}
```

## 5. Tick Intervals (The Compromise)
If you absolutely MUST use `Tick` (e.g., a custom interpolation), stagger it so 1,000 actors don't tick on the same frame.

```cpp
void AMyActor::BeginPlay()
{
    // Tick only 2 times a second instead of 60.
    SetActorTickInterval(0.5f);
}
```

## Verification Checklist
- [ ] `PrimaryActorTick.bCanEverTick = false` is the default state for all new classes.
- [ ] State checks (Health < 0, Distance < X) are moved to Delegates or Overlap events.
- [ ] Logic requiring regular updates uses `GetWorldTimerManager().SetTimer()`.
- [ ] Unavoidable `Tick` functions use `SetActorTickInterval()` to reduce frequency.
