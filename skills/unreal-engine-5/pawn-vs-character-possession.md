---
name: pawn-vs-character-possession
description: Architectural differences between APawn and ACharacter. Rules for possession, root components, and movement replication.
---

# Pawn vs. Character & Possession (UE5 Expert)

## Activation / When to Use
- Mandatory when creating controllable entities.
- Trigger when deciding the base class for a player avatar, vehicle, or AI unit.

## 1. APawn (The Base Body)
The most basic controllable Actor.
- **Root Component**: You must define your own (e.g., `UBoxComponent` or `USphereComponent`).
- **Movement**: You must write your own movement logic (e.g., using `UFloatingPawnMovement` or custom physics).
- **Use Case**: Vehicles, Drones, Stationary Turrets, Simple Flying AI.

## 2. ACharacter (The Bipedal Body)
A specialized subclass of `APawn` tailored for vertical, bipedal entities.
- **Built-in Components**: Comes with a `UCapsuleComponent` (Root), `USkeletalMeshComponent`, and crucially, the `UCharacterMovementComponent` (CMC).
- **Networked Movement**: The CMC includes highly advanced, built-in client-side prediction and server reconciliation for walking, jumping, and swimming.
- **Use Case**: Humanoids, FPS/TPS Avatars, Complex Bipedal AI.

## 3. Possession Architecture
A `Controller` (Player or AI) possesses a Pawn to drive it.
- **Server Authority**: `Possess()` and `UnPossess()` MUST be called on the Server.
- **OnPossess Hook**: Override `PossessedBy(AController* NewController)` on the server or `OnRep_Controller()` on clients to detect when a mind enters the body.

## 4. Impact on Safety
- **Network Desync**: If an agent attempts to build custom replicated movement on an `APawn` without utilizing UE's prediction algorithms, the movement will stutter wildly under latency. Always use `ACharacter` if bipedal movement is required.
- **Component Access**: Casting a Pawn to a Character to access the `GetCharacterMovement()` component is extremely common but crashes if not checked safely.

## Common Mistakes (BAD vs GOOD)

**BAD (Client Possession)**:
```cpp
// Called in a Client RPC or locally
void AMyPlayerController::Client_SwitchVehicle(APawn* NewVehicle) {
    Possess(NewVehicle); // SILENT FAILURE: Only the server can change possession.
}
```

**GOOD (Server Possession)**:
```cpp
// Called on the Server
void AMyPlayerController::Server_SwitchVehicle_Implementation(APawn* NewVehicle) {
    if (HasAuthority() && NewVehicle) {
        Possess(NewVehicle); // SAFE
    }
}
```

**BAD (Using ACharacter for a Car)**:
```text
Inheriting `ACharacter` for a vehicle forces a vertical capsule collision and bipedal movement logic, breaking physics simulations.
```

**GOOD (Choosing APawn for non-bipeds)**:
```text
Inherit `APawn` for a car. Use a Box Component and a `UWheeledVehicleMovementComponent`.
```

## Verification Checklist
- [ ] `ACharacter` is strictly reserved for entities needing bipedal predicted movement.
- [ ] `Possess()` is called exclusively on the Server.
- [ ] Custom Pawns define a valid collision component as their `RootComponent`.
- [ ] Replicated movement on `APawn` uses an explicit movement component or manual interpolation.
