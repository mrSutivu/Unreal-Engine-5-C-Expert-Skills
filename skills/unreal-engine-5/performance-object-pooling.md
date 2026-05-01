---
name: performance-object-pooling
description: Replacing SpawnActor and Destroy with Object Pooling. Disabling Tick, Collision, and Visibility to reuse memory allocations.
---

# Object Pooling (UE5 Expert)

## Activation / When to Use
- Mandatory when dealing with Projectiles, Particles, Floating Combat Text, or rapidly spawning/dying enemies.
- Trigger when `GetWorld()->SpawnActor` and `Actor->Destroy` appear inside rapid fire inputs or heavy loops.

## 1. The Cost of Spawning
`SpawnActor` is incredibly expensive. It allocates memory, registers components, runs constructors, registers with the network, and updates the physics scene. `Destroy` triggers Garbage Collection sweeps.
Doing this 600 times a second for a Minigun destroys performance.

## 2. Object Pooling (The Solution)
Instead of destroying an actor, you hide it and disable its processing. When you need a new actor, you find a hidden one, move it, and unhide it.

### Step 1: Deactivation (The "Destroy" Replacement)
When a projectile hits a wall, turn it "Off".

```cpp
void AMyProjectile::Deactivate()
{
    // 1. Hide Visuals
    SetActorHiddenInGame(true);
    
    // 2. Stop Processing
    SetActorTickEnabled(false);
    
    // 3. Disable Physics/Collision
    CollisionComp->SetCollisionEnabled(ECollisionEnabled::NoCollision);
    
    // 4. Stop Movement
    ProjectileMovementComp->StopMovementImmediately();
    
    // 5. Mark as available for the Pool
    bIsActive = false;
}
```

### Step 2: Activation (The "Spawn" Replacement)
When the gun fires, grab an inactive projectile from the pool array.

```cpp
void AMyProjectile::ActivateAt(FVector Location, FRotator Rotation)
{
    // 1. Move to barrel
    SetActorLocationAndRotation(Location, Rotation);
    
    // 2. Enable Physics/Collision
    CollisionComp->SetCollisionEnabled(ECollisionEnabled::QueryOnly);
    
    // 3. Show Visuals
    SetActorHiddenInGame(false);
    
    // 4. Start Processing
    SetActorTickEnabled(true);
    
    // 5. Fire!
    ProjectileMovementComp->Velocity = Rotation.Vector() * Speed;
    
    bIsActive = true;
}
```

## 3. The Pool Manager
The manager pre-spawns a chunk of actors at `BeginPlay` (e.g., 50 bullets hidden under the map).

```cpp
AMyProjectile* AWeapon::GetPooledProjectile()
{
    // Find first inactive projectile
    for (AMyProjectile* Proj : ProjectilePool)
    {
        if (!Proj->bIsActive) return Proj;
    }
    
    // Pool is empty! Dynamic expansion (Optional)
    AMyProjectile* NewProj = GetWorld()->SpawnActor<AMyProjectile>(...);
    NewProj->Deactivate();
    ProjectilePool.Add(NewProj);
    return NewProj;
}
```

## 4. Impact on Safety & Networking
- **Replication Nightmare**: Object pooling replicated Actors (like enemies) is extremely complex. When an actor is "deactivated", its dormant state replicates to clients. You must manually force clients to snap the actor to the new location upon reactivation before enabling visibility to prevent "teleportation streaks".
- **Rule of Thumb**: Pool visual effects, UI, and simple projectiles. Avoid pooling complex networked Characters unless absolutely necessary for performance.

## Verification Checklist
- [ ] Rapid-fire entities (bullets, damage text) are pre-spawned and stored in a `TArray`.
- [ ] Deactivation sets `SetActorHiddenInGame(true)`, `SetActorTickEnabled(false)`, and disables collision.
- [ ] `SpawnActor` is NEVER used during rapid gameplay loops.
