---
name: anim-montage-cpp
description: Playing AnimMontages from C++ and binding delegates (FOnMontageEnded) to detect completion.
---

# AnimMontage Execution in C++ (UE5 Expert)

## Activation / When to Use
- Mandatory when triggering complex, full-body or upper-body animations (Attacks, Reloads, Interactions).
- Trigger when you need to know exactly when an animation finishes to execute gameplay logic.

## 1. Core Principles
An `UAnimMontage` is an asset containing animation sequences, sections, and notifies. It is played via the `UAnimInstance`.

## 2. Implementation Syntax

### Playing the Montage
Use `PlayAnimMontage` on the Character (which routes to the AnimInstance) or call it on the AnimInstance directly.

```cpp
void AMyCharacter::PerformAttack()
{
    if (AttackMontage)
    {
        // Play the montage and get the duration
        float Duration = PlayAnimMontage(AttackMontage, 1.0f);
        
        // Alternatively, specific section:
        // PlayAnimMontage(AttackMontage, 1.0f, FName("Combo2"));
    }
}
```

### Binding to Completion (The Delegate)
To execute code when the animation ends or is interrupted, bind to `OnMontageEnded`.

```cpp
void AMyCharacter::PerformAttack()
{
    if (AttackMontage)
    {
        UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
        if (AnimInstance)
        {
            // 1. Play
            AnimInstance->Montage_Play(AttackMontage);

            // 2. Bind Delegate
            FOnMontageEnded EndDelegate;
            EndDelegate.BindUObject(this, &AMyCharacter::OnAttackMontageEnded);
            AnimInstance->Montage_SetEndDelegate(EndDelegate, AttackMontage);
        }
    }
}

// Callback signature must match EXACTLY
void AMyCharacter::OnAttackMontageEnded(UAnimMontage* Montage, bool bInterrupted)
{
    if (bInterrupted) {
        // Handle stagger / cancel logic
    } else {
        // Handle successful completion
    }
}
```

## 3. Impact on Safety
- **Interruption Handling**: The `bInterrupted` flag is critical. If a player is stunned during an attack montage, the montage ends. If you don't check `bInterrupted`, your code might incorrectly apply damage or spawn a projectile.
- **Zombie Callbacks**: `BindUObject` safely unbinds if the Character is destroyed, preventing memory access violations if the AnimInstance outlives the Actor.

## Verification Checklist
- [ ] `PlayAnimMontage` or `Montage_Play` is used (never basic `PlayAnimation` for gameplay logic).
- [ ] `FOnMontageEnded` delegate is bound if follow-up logic is required.
- [ ] The callback function checks the `bInterrupted` flag to prevent logic errors.
