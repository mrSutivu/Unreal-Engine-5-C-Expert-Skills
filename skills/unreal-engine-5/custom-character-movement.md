---
name: custom-character-movement
description: Subclassing UCharacterMovementComponent to create replicated, predicted movement modes (Dash, WallRun, Grapple).
---

# Custom Character Movement (UE5 Expert)

## Activation / When to Use
- Mandatory when adding new movement mechanics (Dash, WallRun, Slide, Grapple) to a multiplayer game.
- Trigger when you need network prediction and server reconciliation for physics.

## 1. Core Principles
You CANNOT just add `AddActorWorldOffset` in `Tick` for a multiplayer game. The server will see the player teleporting and rubberband them back.
You must subclass `UCharacterMovementComponent` (CMC) and use `FSavedMove_Character`.

## 2. Implementation Syntax

### 1. Subclass the CMC
```cpp
#pragma once
#include "GameFramework/CharacterMovementComponent.h"
#include "MyMovementComponent.generated.h"

UCLASS()
class MYPROJECT_API UMyMovementComponent : public UCharacterMovementComponent
{
    GENERATED_BODY()

public:
    // Define a custom movement flag
    uint8 bWantsToDash : 1;

    // Triggered by the Character
    void DoDash();

    // Core Overrides for Prediction
    virtual void UpdateFromCompressedFlags(uint8 Flags) override;
    virtual class FNetworkPredictionData_Client* GetPredictionData_Client() const override;
    virtual void OnMovementUpdated(float DeltaSeconds, const FVector& OldLocation, const FVector& OldVelocity) override;
};
```

### 2. The Saved Move Struct (Client Prediction)
This struct packages the player's input and flags to send to the Server.

```cpp
class FSavedMove_MyMovement : public FSavedMove_Character
{
public:
    uint8 bSavedWantsToDash : 1;

    virtual void Clear() override;
    virtual uint8 GetCompressedFlags() const override;
    virtual void SetMoveFor(ACharacter* C, float InDeltaTime, FVector const& NewAccel, class FNetworkPredictionData_Client_Character& ClientData) override;
    virtual void PrepMoveFor(ACharacter* C) override;
};
```

### 3. Flag Compression (Sending to Server)
Map your custom boolean to one of the 4 free flags UE provides.

```cpp
uint8 FSavedMove_MyMovement::GetCompressedFlags() const
{
    uint8 Result = Super::GetCompressedFlags();
    if (bSavedWantsToDash)
    {
        Result |= FLAG_Custom_0; // Tells server "Flag 0 is active"
    }
    return Result;
}

void UMyMovementComponent::UpdateFromCompressedFlags(uint8 Flags)
{
    Super::UpdateFromCompressedFlags(Flags);
    bWantsToDash = (Flags & FLAG_Custom_0) != 0; // Server unpacks it
}
```

### 4. Linking to Character
Tell your `ACharacter` to use your custom CMC in its constructor.

```cpp
AMyCharacter::AMyCharacter(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer.SetDefaultSubobjectClass<UMyMovementComponent>(ACharacter::CharacterMovementComponentName))
{
}
```

## 3. Impact on Safety
- **Rubberbanding**: Properly setting up `FSavedMove` guarantees that if the client dashes, the server processes the exact same dash logic simultaneously, ensuring they stay perfectly synchronized.
- **Server Authority**: The server still runs `OnMovementUpdated`. If a hacked client tries to dash while stunned, the server ignores the flag and corrects the client's position.

## Verification Checklist
- [ ] Subclasses `UCharacterMovementComponent`.
- [ ] Custom struct inherits from `FSavedMove_Character`.
- [ ] Bools are packed into `FLAG_Custom_0` through `3`.
- [ ] `SetDefaultSubobjectClass` is used in the Character constructor.
