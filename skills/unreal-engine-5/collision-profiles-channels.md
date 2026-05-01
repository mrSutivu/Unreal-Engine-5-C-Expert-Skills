---
name: collision-profiles-channels
description: C++ handling of Physics and Collisions. Utilizing ECC_GameTraceChannel, ECR_Block, and dynamic collision responses.
---

# Collision Profiles & Channels (UE5 Expert)

## Activation / When to Use
- Mandatory when spawning physical objects, configuring damage hitboxes, or setting up trigger volumes.
- Trigger when dealing with `UPrimitiveComponent` (Capsules, Boxes, Meshes).

## 1. Core Principles
Unreal handles collision through two main concepts:
- **Object Type**: What am I? (e.g., `WorldStatic`, `Pawn`, `PhysicsBody`).
- **Collision Responses**: How do I react to other Object Types? (Block, Overlap, Ignore).

## 2. Setting Up Collision in C++

### Constructors (Default State)
Always configure the base collision rules in the constructor.

```cpp
AMyActor::AMyActor()
{
    BoxComp = CreateDefaultSubobject<UBoxComponent>(TEXT("BoxComp"));
    RootComponent = BoxComp;

    // 1. Enable Physics/Collision
    BoxComp->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);

    // 2. Set Object Type
    BoxComp->SetCollisionObjectType(ECC_WorldDynamic);

    // 3. Set Responses
    BoxComp->SetCollisionResponseToAllChannels(ECR_Ignore); // Ignore everything by default
    BoxComp->SetCollisionResponseToChannel(ECC_Pawn, ECR_Overlap); // Except Pawns, which we overlap
    BoxComp->SetCollisionResponseToChannel(ECC_WorldStatic, ECR_Block); // And walls, which we hit
}
```

## 3. Custom Trace Channels
In Project Settings, you can create custom channels (e.g., "WeaponTrace"). In C++, these map to `ECC_GameTraceChannel1` through `18`.

```cpp
// Better to use a macro to name them clearly
#define ECC_WeaponTrace ECC_GameTraceChannel1

// Usage
MeshComp->SetCollisionResponseToChannel(ECC_WeaponTrace, ECR_Block);
```

## 4. Collision Enabled Types (`ECollisionEnabled`)
Optimizing the collision type saves massive CPU overhead.
- `NoCollision`: Cheap. Visual only.
- `QueryOnly`: Used for Traces (Raycasts) and Overlaps. Cannot be physically pushed. (Best for triggers).
- `PhysicsOnly`: Used for Ragdolls or falling debris. Doesn't respond to Traces.
- `QueryAndPhysics`: Most expensive. Used for Characters and interactive physics props.

## 5. Impact on Safety
- **Performance Leak**: Setting every object to `QueryAndPhysics` and blocking everything will destroy CPU performance. Use `QueryOnly` and `ECR_Ignore` heavily.
- **Overlap Failures**: An overlap event requires BOTH objects to have `ECollisionEnabled::QueryOnly` (or higher) AND BOTH must have their response to each other's Object Type set to `ECR_Overlap`.

## Common Mistakes (BAD vs GOOD)

**BAD (Unoptimized Default)**:
```cpp
BoxComp->SetCollisionProfileName(TEXT("BlockAll")); // Blocks everything, expensive!
```

**GOOD (Surgical Channels)**:
```cpp
BoxComp->SetCollisionEnabled(ECollisionEnabled::QueryOnly);
BoxComp->SetCollisionResponseToAllChannels(ECR_Ignore);
BoxComp->SetCollisionResponseToChannel(ECC_Pawn, ECR_Overlap); // Cheap, clean trigger
```

## Verification Checklist
- [ ] `ECollisionEnabled` is set to the lowest required tier (QueryOnly vs Physics).
- [ ] Custom trace channels use macro defines (`#define ECC_WeaponTrace ECC_GameTraceChannel1`).
- [ ] Overlap and Block configurations are explicitly defined rather than relying on preset strings if custom behavior is needed.
