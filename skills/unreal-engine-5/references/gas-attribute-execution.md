---
name: gas-attribute-execution
description: Complete C++ boilerplate for UAttributeSet and UGameplayEffectExecutionCalculation in the Gameplay Ability System.
---

# GAS Attribute & Execution Template (UE5 Expert Reference)

## Activation / When to Use
- Trigger when you need the EXACT boilerplate for a networked `UAttributeSet` with clamping and RepNotifies.
- Trigger when building a complex Damage Execution Calculation class.

## 1. The Ultimate UAttributeSet Template

### Header (`MyAttributeSet.h`)
```cpp
#pragma once
#include "CoreMinimal.h"
#include "AttributeSet.h"
#include "AbilitySystemComponent.h"
#include "MyAttributeSet.generated.h"

// Required macro for getters/setters
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
    UMyAttributeSet();

    // 1. Replicated Attribute (Health)
    UPROPERTY(BlueprintReadOnly, Category = "Stats", ReplicatedUsing = OnRep_Health)
    FGameplayAttributeData Health;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, Health);

    UPROPERTY(BlueprintReadOnly, Category = "Stats", ReplicatedUsing = OnRep_MaxHealth)
    FGameplayAttributeData MaxHealth;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, MaxHealth);

    // 2. Meta Attribute (Damage - NEVER REPLICATED)
    UPROPERTY(BlueprintReadOnly, Category = "Stats")
    FGameplayAttributeData Damage;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, Damage);

    // 3. Core Overrides
    virtual void PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue) override;
    virtual void PostGameplayEffectExecute(const struct FGameplayEffectModCallbackData& Data) override;
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

protected:
    // 4. RepNotifies
    UFUNCTION()
    virtual void OnRep_Health(const FGameplayAttributeData& OldHealth);

    UFUNCTION()
    virtual void OnRep_MaxHealth(const FGameplayAttributeData& OldMaxHealth);
};
```

### Source (`MyAttributeSet.cpp`)
```cpp
#include "MyAttributeSet.h"
#include "Net/UnrealNetwork.h"

UMyAttributeSet::UMyAttributeSet() {}

void UMyAttributeSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, Health, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, MaxHealth, COND_None, REPNOTIFY_Always);
}

void UMyAttributeSet::OnRep_Health(const FGameplayAttributeData& OldHealth) {
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyAttributeSet, Health, OldHealth);
}

void UMyAttributeSet::OnRep_MaxHealth(const FGameplayAttributeData& OldMaxHealth) {
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyAttributeSet, MaxHealth, OldMaxHealth);
}

void UMyAttributeSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
    Super::PreAttributeChange(Attribute, NewValue);

    // Hard Clamping
    if (Attribute == GetHealthAttribute()) {
        NewValue = FMath::Clamp(NewValue, 0.0f, GetMaxHealth());
    }
}

void UMyAttributeSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    Super::PostGameplayEffectExecute(Data);

    if (Data.EvaluatedData.Attribute == GetDamageAttribute())
    {
        // Extract Damage
        float LocalDamage = GetDamage();
        SetDamage(0.0f); // Reset the meta attribute instantly

        if (LocalDamage > 0.0f)
        {
            // Apply it to health
            float NewHealth = GetHealth() - LocalDamage;
            SetHealth(FMath::Clamp(NewHealth, 0.0f, GetMaxHealth()));
            
            // Check Death here if Health <= 0.0f
        }
    }
}
```

## 2. The Execution Calculation Template

### Header (`MyDamageExecution.h`)
```cpp
#pragma once
#include "GameplayEffectExecutionCalculation.h"
#include "MyDamageExecution.generated.h"

UCLASS()
class UMyDamageExecution : public UGameplayEffectExecutionCalculation
{
    GENERATED_BODY()

public:
    UMyDamageExecution();
    virtual void Execute_Implementation(const FGameplayEffectCustomExecutionParameters& ExecutionParams, OUT FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const override;
};
```

### Source (`MyDamageExecution.cpp`)
```cpp
#include "MyDamageExecution.h"
#include "MyAttributeSet.h"

// Define Capture Struct
struct FDamageStatics
{
    DECLARE_ATTRIBUTE_CAPTUREDEF(MaxHealth);
    
    FDamageStatics()
    {
        // Capture the Target's MaxHealth. Snapshot = false (Get current value)
        DEFINE_ATTRIBUTE_CAPTUREDEF(UMyAttributeSet, MaxHealth, Target, false);
    }
};

static const FDamageStatics& DamageStatics()
{
    static FDamageStatics Statics;
    return Statics;
}

UMyDamageExecution::UMyDamageExecution()
{
    RelevantAttributesToCapture.Add(DamageStatics().MaxHealthDef);
}

void UMyDamageExecution::Execute_Implementation(const FGameplayEffectCustomExecutionParameters& ExecutionParams, OUT FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const
{
    const FGameplayEffectSpec& Spec = ExecutionParams.GetOwningSpec();
    FAggregatorEvaluateParameters EvalParams;
    EvalParams.SourceTags = Spec.CapturedSourceTags.GetAggregatedTags();
    EvalParams.TargetTags = Spec.CapturedTargetTags.GetAggregatedTags();

    // 1. Read Captured Attribute
    float TargetMaxHealth = 0.0f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(DamageStatics().MaxHealthDef, EvalParams, TargetMaxHealth);

    // 2. Read SetByCaller Value (e.g., Base Damage from ability)
    float BaseDamage = Spec.GetSetByCallerMagnitude(FGameplayTag::RequestGameplayTag(FName("Data.Damage")), false, 0.0f);

    // 3. Formula: 10% of Max Health + Base Damage
    float TotalDamage = BaseDamage + (TargetMaxHealth * 0.1f);

    // 4. Output to Damage Meta Attribute
    if (TotalDamage > 0.0f)
    {
        OutExecutionOutput.AddOutputModifier(FGameplayModifierEvaluatedData(UMyAttributeSet::GetDamageAttribute(), EGameplayModOp::Additive, TotalDamage));
    }
}
```
