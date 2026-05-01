---
name: enhanced-input-system
description: Expert setup for UE 5.1+ Enhanced Input System in C++. Best practices for InputMappingContexts, Action binding, and Data Assets.
---

# Enhanced Input System Mapping (UE5 Expert)

## Activation / When to Use
- Mandatory for all player input handling in UE 5.1+.
- Deprecates the old `InputCore` axis/action mappings.

## 1. Core Architecture (The Data Asset Pattern)
Do not hardcode `UInputAction` pointers directly in the Character. Create an `UDataAsset` to hold them.

### Input Config Asset (.h)
```cpp
UCLASS()
class UMyInputConfig : public UDataAsset
{
    GENERATED_BODY()
public:
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TObjectPtr<UInputAction> InputMove;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TObjectPtr<UInputAction> InputJump;
};
```

## 2. Character Setup

### Header
```cpp
UPROPERTY(EditDefaultsOnly, Category = "Input")
TObjectPtr<UInputMappingContext> DefaultMappingContext;

UPROPERTY(EditDefaultsOnly, Category = "Input")
TObjectPtr<UMyInputConfig> InputConfig;

void Move(const FInputActionValue& Value);
```

### Binding (.cpp)
Bind actions in `SetupPlayerInputComponent`. Use `CastChecked`.
```cpp
void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    UEnhancedInputComponent* EnhancedInputComponent = CastChecked<UEnhancedInputComponent>(PlayerInputComponent);

    if (InputConfig) {
        EnhancedInputComponent->BindAction(InputConfig->InputMove, ETriggerEvent::Triggered, this, &AMyCharacter::Move);
        EnhancedInputComponent->BindAction(InputConfig->InputJump, ETriggerEvent::Started, this, &ACharacter::Jump);
    }
}
```

## 3. Applying the Mapping Context
Apply context via the `UEnhancedInputLocalPlayerSubsystem`. Best done in `PawnClientRestart` for multiplayer safety, but `BeginPlay` works for single-player.

```cpp
void AMyCharacter::PawnClientRestart()
{
    Super::PawnClientRestart();
    if (APlayerController* PC = Cast<APlayerController>(GetController())) {
        if (UEnhancedInputLocalPlayerSubsystem* Subsystem = ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PC->GetLocalPlayer())) {
            Subsystem->AddMappingContext(DefaultMappingContext, 0); // 0 = Priority
        }
    }
}
```

## 4. Reading Values
`FInputActionValue` holds bools, float1D, Vector2D, or Vector3D.
```cpp
void AMyCharacter::Move(const FInputActionValue& Value)
{
    FVector2D MovementVector = Value.Get<FVector2D>();
    AddMovementInput(GetActorForwardVector(), MovementVector.Y);
    AddMovementInput(GetActorRightVector(), MovementVector.X);
}
```

## 5. Impact on Safety
- **Modular Contexts**: Swapping MappingContexts (e.g., Walking to Driving) via the Subsystem prevents complex `if/else` states in the Character.
- **Modifier Offloading**: Deadzones and Axis Inversions are handled in the Asset, keeping C++ logic pure and safe.

## Verification Checklist
- [ ] `EnhancedInput` module added to `Build.cs`.
- [ ] `UDataAsset` used to store `UInputAction` references.
- [ ] `CastChecked<UEnhancedInputComponent>` used in Setup.
- [ ] Context added via `UEnhancedInputLocalPlayerSubsystem`.
- [ ] `Triggered` used for axes, `Started` used for one-shot actions.
