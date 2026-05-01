---
name: gas-uattributeset
description: Architecture of UAttributeSet in the Gameplay Ability System. Implementing ATTRIBUTE_ACCESSORS and clamping values securely.
---

# GAS: UAttributeSet Architecture (UE5 Expert)

## Activation / When to Use
- Mandatory when integrating the Gameplay Ability System (GAS) for health, mana, stamina, or RPG stats.
- Trigger when you need centralized, replicated variables that can be modified by GameplayEffects.

## 1. Core Principles
The `UAttributeSet` holds `FGameplayAttributeData`. It dictates the base values, current values, and clamping rules (e.g., Health cannot drop below 0).

## 2. Implementation Syntax

### Header (.h)
You MUST use the `ATTRIBUTE_ACCESSORS` macro to generate the necessary getter/setter/init boilerplates.

```cpp
#pragma once
#include "AttributeSet.h"
#include "AbilitySystemComponent.h"
#include "MyAttributeSet.generated.h"

// Boilerplate macro
#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
    GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)

UCLASS()
class MYPROJECT_API UMyAttributeSet : public UAttributeSet
{
    GENERATED_BODY()

public:
    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Health)
    FGameplayAttributeData Health;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, Health);

    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxHealth)
    FGameplayAttributeData MaxHealth;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, MaxHealth);

protected:
    UFUNCTION()
    virtual void OnRep_Health(const FGameplayAttributeData& OldHealth);

    UFUNCTION()
    virtual void OnRep_MaxHealth(const FGameplayAttributeData& OldMaxHealth);
};
```

### Source (.cpp) (Replication & Clamping)
Register replication in `GetLifetimeReplicatedProps`. Enforce clamping in `PreAttributeChange`.

```cpp
#include "Net/UnrealNetwork.h"

void UMyAttributeSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, Health, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, MaxHealth, COND_None, REPNOTIFY_Always);
}

// Boilerplate RepNotifies
void UMyAttributeSet::OnRep_Health(const FGameplayAttributeData& OldHealth) {
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyAttributeSet, Health, OldHealth);
}
void UMyAttributeSet::OnRep_MaxHealth(const FGameplayAttributeData& OldMaxHealth) {
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyAttributeSet, MaxHealth, OldMaxHealth);
}

// Safety Clamping (Runs BEFORE modifier is applied)
void UMyAttributeSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
    Super::PreAttributeChange(Attribute, NewValue);

    if (Attribute == GetHealthAttribute()) {
        NewValue = FMath::Clamp(NewValue, 0.0f, GetMaxHealth());
    }
}
```

## 3. PostGameplayEffectExecute
To handle "Damage" safely, use a Meta Attribute (an attribute that isn't replicated, like `Damage`) and process it here.

```cpp
void UMyAttributeSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    Super::PostGameplayEffectExecute(Data);

    if (Data.EvaluatedData.Attribute == GetDamageAttribute())
    {
        // Apply damage to health
        float LocalDamageDone = GetDamage();
        SetDamage(0.0f); // Reset meta attribute
        
        float NewHealth = GetHealth() - LocalDamageDone;
        SetHealth(FMath::Clamp(NewHealth, 0.0f, GetMaxHealth()));
        
        // Trigger Death if 0
    }
}
```

## 4. Impact on Safety
- **Clamping**: `PreAttributeChange` ensures that no Gameplay Effect, no matter how badly configured by a designer, can push Health above MaxHealth or below 0.
- **Meta Attributes**: Processing `Damage` rather than modifying `Health` directly allows for complex armor/resistance calculations inside `PostGameplayEffectExecute` or Execution Calculations.

## Verification Checklist
- [ ] `ATTRIBUTE_ACCESSORS` macro is used for every `FGameplayAttributeData`.
- [ ] `REPNOTIFY_Always` is used in `DOREPLIFETIME_CONDITION_NOTIFY` for attributes.
- [ ] `GAMEPLAYATTRIBUTE_REPNOTIFY` is used inside the `OnRep_` functions.
- [ ] Hard clamps are enforced in `PreAttributeChange`.
