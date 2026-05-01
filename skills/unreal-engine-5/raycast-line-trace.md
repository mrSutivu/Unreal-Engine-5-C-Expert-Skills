---
name: raycast-line-trace
description: Executing collision queries (LineTraceSingleByChannel, SweepMultiByObjectType). Extracting and validating FHitResult data.
---

# Raycasts & Line Traces (UE5 Expert)

## Activation / When to Use
- Mandatory for hitscan weapons, line of sight checks, foot placement (IK), and interaction systems.
- Trigger when you need to detect physical objects along a path.

## 1. Core Principles
Traces (Raycasts) project a line (or shape) through the physics world.
- **ByChannel**: Uses visibility/camera/custom channels (e.g., "Is there a wall here?").
- **ByObjectType**: Looks for specific physical types (e.g., "Find all Pawns in this radius").
- **Single vs Multi**: Returns the first hit vs returns all hits along the path.

## 2. Implementation Syntax

### Simple Line Trace
```cpp
void AMyCharacter::PerformHitscan()
{
    FVector Start = GetActorLocation();
    FVector End = Start + (GetActorForwardVector() * 5000.0f);

    FHitResult HitResult;
    FCollisionQueryParams QueryParams;
    QueryParams.AddIgnoredActor(this); // Ignore self

    // Perform Trace
    bool bHit = GetWorld()->LineTraceSingleByChannel(
        HitResult,
        Start,
        End,
        ECC_Visibility, // Trace Channel
        QueryParams
    );

    if (bHit && HitResult.GetActor())
    {
        UE_LOG(LogTemp, Log, TEXT("Hit: %s"), *HitResult.GetActor()->GetName());
    }
}
```

### Shape Sweep (e.g., Sphere Trace)
Sweeps a 3D shape, much better for melee attacks or thick projectiles.

```cpp
FCollisionShape SphereShape = FCollisionShape::MakeSphere(50.0f);
TArray<FHitResult> HitResults;

bool bHit = GetWorld()->SweepMultiByObjectType(
    HitResults,
    Start,
    End,
    FQuat::Identity,
    FCollisionObjectQueryParams(ECC_Pawn), // Only look for Pawns
    SphereShape,
    QueryParams
);
```

## 3. Parsing FHitResult
The `FHitResult` struct contains massive amounts of data about the impact.
- `Hit.GetActor()`: The Actor hit (can be null if hit world geometry!).
- `Hit.GetComponent()`: The specific primitive component hit.
- `Hit.Location`: The point in world space where the trace stopped.
- `Hit.ImpactPoint`: The exact point on the surface of the object.
- `Hit.ImpactNormal`: The direction the surface is facing (perfect for spawning decals).
- `Hit.BoneName`: The specific bone hit (Requires `bReturnPhysicalMaterial = true` in query params for accuracy).

## 4. Impact on Safety & Debugging
- **Null Safety**: You MUST verify `Hit.GetActor()` is valid before calling functions on it. Raycasts can hit BSP BSP brushes or unowned geometry which return null Actors.
- **Visual Debugging**: Use `DrawDebugLine` (requires `#include "DrawDebugHelpers.h"`) to visualize the trace during development.

## Verification Checklist
- [ ] `AddIgnoredActor(this)` is called to prevent tracing against oneself.
- [ ] `LineTraceSingleByChannel` used for precise checks (Hitscan, Line of sight).
- [ ] `SweepMulti` used for voluminous checks (Melee, Explosions).
- [ ] `HitResult.GetActor()` is checked for null before access.
