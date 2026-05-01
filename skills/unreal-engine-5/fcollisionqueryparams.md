---
name: fcollisionqueryparams
description: Configuring FCollisionQueryParams to optimize physical queries. Handling bTraceComplex, Ignored Actors, and Physical Materials.
---

# FCollisionQueryParams Configuration (UE5 Expert)

## Activation / When to Use
- Mandatory whenever calling a collision query (`LineTrace`, `Sweep`, `Overlap`) in C++.
- Trigger when you need to filter out specific actors or extract advanced data (like Physical Materials).

## 1. Core Principles
`FCollisionQueryParams` dictates *how* the physics engine executes the trace. Proper configuration drastically improves performance and accuracy.

## 2. Key Properties

### 1. Tag (Debugging)
Give your trace a name. This shows up in the Unreal Insights / Visual Logger profiler.
```cpp
FCollisionQueryParams Params(FName("WeaponTrace"), false, this);
```

### 2. bTraceComplex (Accuracy vs Performance)
- **False (Default)**: Traces against the *Simple Collision* (Capsules, Boxes, convex hulls). Extremely fast. Used for movement and generic queries.
- **True**: Traces against the *Complex Collision* (per-poly mesh). Expensive. Use ONLY when pixel-perfect accuracy is required (e.g., Sniper bullet hitting a tiny gap in a fence).

```cpp
Params.bTraceComplex = true;
```

### 3. bReturnPhysicalMaterial
- **False (Default)**: `HitResult.PhysMaterial` will be null.
- **True**: Forces the engine to return the Physical Material assigned to the surface. Required for determining footstep sounds or bullet decal types (Dirt, Metal, Flesh).

```cpp
Params.bReturnPhysicalMaterial = true;
```

### 4. Ignoring Actors
Critically important to prevent your gun from hitting your own collision capsule.

```cpp
Params.AddIgnoredActor(this);
Params.AddIgnoredActor(MyEquippedWeapon);

// Or ignore an array of actors
TArray<AActor*> IgnoredActors;
Params.AddIgnoredActors(IgnoredActors);
```

## 3. Impact on Safety
- **Self-Harm**: Failing to ignore `this` causes hitscan weapons to detonate instantly inside the player's face.
- **Performance Ruin**: Setting `bTraceComplex = true` on an AI Sight perception sweep running every frame will destroy server framerates. Complex traces must be surgical.

## Common Mistakes (BAD vs GOOD)

**BAD (Unconfigured Query)**:
```cpp
FCollisionQueryParams Params;
GetWorld()->LineTraceSingleByChannel(Hit, Start, End, ECC_Visibility, Params); // Hits self!
```

**GOOD (Configured Query)**:
```cpp
FCollisionQueryParams Params(SCENE_QUERY_STAT(Hitscan), true, this); // Named, Complex, Ignores Self
```

## Verification Checklist
- [ ] `AddIgnoredActor(this)` is explicitly called on all weapon and sight traces.
- [ ] `bTraceComplex` is `false` unless per-polygon accuracy is strictly required.
- [ ] `bReturnPhysicalMaterial` is `true` ONLY if surface types are needed for logic/VFX.
- [ ] Trace is named using `SCENE_QUERY_STAT` for profiling.
