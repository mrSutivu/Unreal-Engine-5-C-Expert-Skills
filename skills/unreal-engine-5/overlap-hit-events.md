---
name: overlap-hit-events
description: Safely binding C++ dynamic delegates to OnComponentBeginOverlap and OnComponentHit.
---

# Overlap and Hit Events (UE5 Expert)

## Activation / When to Use
- Mandatory for trigger zones, damage application from projectiles, and physical interactions.
- Trigger when you need C++ logic to execute when two components touch or collide.

## 1. Core Principles
Collision events in UE5 are broadcast via Dynamic Multicast Delegates.
- **Hit Event**: Requires `Simulation Generates Hit Events` to be true. Triggered by a solid collision (Block).
- **Overlap Event**: Requires `Generate Overlap Events` to be true. Triggered when two volumes intersect (Overlap).

## 2. Implementation Syntax

### Header (.h)
The callback functions MUST be marked with `UFUNCTION()` or the binding will fail silently.

```cpp
UCLASS()
class MYPROJECT_API AMyTrigger : public AActor
{
    GENERATED_BODY()

protected:
    UPROPERTY(VisibleAnywhere)
    class UBoxComponent* TriggerBox;

    // MUST HAVE UFUNCTION()
    UFUNCTION()
    void OnOverlapBegin(UPrimitiveComponent* OverlappedComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult);

    UFUNCTION()
    void OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit);
};
```

### Source (.cpp) - Binding the Delegates
**CRITICAL**: Do NOT bind dynamic delegates in the constructor. Bind them in `BeginPlay` or `PostInitializeComponents`. Binding in the constructor can cause bugs during hot-reloads and Blueprint compilation.

```cpp
AMyTrigger::AMyTrigger()
{
    TriggerBox = CreateDefaultSubobject<UBoxComponent>(TEXT("TriggerBox"));
    TriggerBox->SetGenerateOverlapEvents(true);
    RootComponent = TriggerBox;
}

void AMyTrigger::BeginPlay()
{
    Super::BeginPlay();

    // Bind Overlap
    TriggerBox->OnComponentBeginOverlap.AddDynamic(this, &AMyTrigger::OnOverlapBegin);
    
    // Bind Hit (if needed)
    // TriggerBox->OnComponentHit.AddDynamic(this, &AMyTrigger::OnHit);
}
```

### Execution Logic
Always validate the `OtherActor` before acting.

```cpp
void AMyTrigger::OnOverlapBegin(UPrimitiveComponent* OverlappedComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
{
    if (OtherActor && OtherActor != this)
    {
        if (AMyCharacter* Player = Cast<AMyCharacter>(OtherActor))
        {
            // Apply logic
        }
    }
}
```

## 3. Impact on Safety
- **Silent Failures**: Missing the `UFUNCTION()` macro is the most common reason overlap events "don't work" in C++. The compiler won't warn you.
- **Infinite Loops**: Failing to check `OtherActor != this` can cause a projectile to hit itself and explode instantly.

## Verification Checklist
- [ ] Callback functions have EXACT matching signatures for `OnComponentBeginOverlap` or `OnComponentHit`.
- [ ] Callback functions are decorated with `UFUNCTION()`.
- [ ] Delegates are bound in `BeginPlay`, NOT the constructor.
- [ ] `OtherActor` validity is checked before execution.
