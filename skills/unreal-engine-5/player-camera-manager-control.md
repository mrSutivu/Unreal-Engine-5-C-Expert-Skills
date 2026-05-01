---
name: player-camera-manager-control
description: Advanced usage of APlayerCameraManager. Handles View Targets, camera blends, and Camera Shakes independently of the Pawn.
---

# Player Camera Manager Control (UE5 Expert)

## Activation / When to Use
- Mandatory when taking camera control away from the player (Cutscenes, execution moves, respawn spectating).
- Trigger when implementing complex Camera Shakes or localized post-processing.

## 1. Core Principles
The `APlayerCameraManager` (PCM) is owned by the `APlayerController`. 
- **Decoupling**: The camera is NOT permanently glued to the `APawn`'s SpringArm. The PCM determines the "View Target" every frame.
- **Default Behavior**: If the PlayerController possesses a Pawn, the PCM automatically sets that Pawn as the View Target.

## 2. Changing the View Target (SetViewTargetWithBlend)
You can detach the camera from the player and blend it to ANY Actor in the world smoothly.

```cpp
void AMyPlayerController::FocusOnObjective(AActor* ObjectiveActor)
{
    if (ObjectiveActor)
    {
        // Blends the camera from current view to the ObjectiveActor over 2.0 seconds
        SetViewTargetWithBlend(ObjectiveActor, 2.0f, EViewTargetBlendFunction::VTBlend_Cubic);
    }
}

void AMyPlayerController::ResetCamera()
{
    // Blends back to the controlled Pawn
    if (APawn* MyPawn = GetPawn())
    {
        SetViewTargetWithBlend(MyPawn, 1.0f);
    }
}
```

## 3. Camera Shakes (UE 5.1+)
UE5 uses `UCameraShakeBase` and the specialized `UCameraShakePattern` (like `UPerlinNoiseCameraShakePattern`).

### Playing a Shake
Shakes are played via the PlayerCameraManager.

```cpp
// Header
UPROPERTY(EditDefaultsOnly)
TSubclassOf<UCameraShakeBase> HitShakeClass;

// Source
void AMyCharacter::TakeDamage()
{
    if (APlayerController* PC = Cast<APlayerController>(GetController()))
    {
        if (APlayerCameraManager* CameraManager = PC->PlayerCameraManager)
        {
            // Plays the shake for this specific player
            CameraManager->StartCameraShake(HitShakeClass, 1.0f);
        }
    }
}
```

## 4. Impact on Safety
- **Actor Destruction Safety**: If the `ViewTarget` actor is destroyed, the PCM automatically falls back safely (usually to the PlayerController's location). It will not crash.
- **Modularity**: By managing camera logic in the PCM or PlayerController, the agent ensures that `APawn` code remains focused purely on movement and stats, reducing God-Class bloat.

## Common Mistakes (BAD vs GOOD)

**BAD (Moving the Camera Component manually)**:
```cpp
// Detaching the camera component from the Pawn to look at something else.
MyCameraComp->DetachFromComponent(...);
MyCameraComp->SetWorldLocation(TargetLoc); // Messy, breaks input logic.
```

**GOOD (Using View Targets)**:
```cpp
// Leaves the Pawn's hierarchy alone. Let the Engine handle the camera view.
PC->SetViewTargetWithBlend(TargetActor, 1.0f);
```

## Verification Checklist
- [ ] `SetViewTargetWithBlend` is used for dramatic camera transitions instead of manipulating the `UCameraComponent` directly.
- [ ] Camera Shakes are implemented via `UCameraShakeBase` classes.
- [ ] Camera Shakes are triggered via `PlayerCameraManager->StartCameraShake`.
