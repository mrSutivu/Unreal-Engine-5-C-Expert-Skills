---
name: gas-execution-calculations
description: Implementing UGameplayEffectExecutionCalculation for complex damage formulas involving Target and Source attributes.
---

# GAS: Execution Calculations (UE5 Expert)

## Activation / When to Use
- Mandatory when a `UGameplayEffect` requires complex math (e.g., `Damage = (SourceAttack * 1.5) - TargetArmor`).
- Trigger when simple `Add`, `Multiply`, or `SetByCaller` Modifiers in the Gameplay Effect are insufficient.

## 1. Core Principles
A `UGameplayEffectExecutionCalculation` (ExecCalc) is a C++ class that captures Attributes from both the Source (Attacker) and the Target (Defender) before applying modifications.

## 2. Implementation Syntax

### Header (.h)
You must define `FGameplayEffectAttributeCaptureDefinition` for every attribute you want to read.

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

### Source (.cpp) - The Capture Struct
A local struct is often used to hold the capture definitions cleanly.

```cpp
#include "MyAttributeSet.h" // Your AttributeSet header

struct FDamageStatics
{
    DECLARE_ATTRIBUTE_CAPTUREDEF(Armor);
    DECLARE_ATTRIBUTE_CAPTUREDEF(AttackPower);
    
    FDamageStatics()
    {
        // Target's Armor (Snapshot = false means grab current value)
        DEFINE_ATTRIBUTE_CAPTUREDEF(UMyAttributeSet, Armor, Target, false);
        // Source's Attack Power (Snapshot = true means grab value at time of effect creation)
        DEFINE_ATTRIBUTE_CAPTUREDEF(UMyAttributeSet, AttackPower, Source, true);
    }
};

static const FDamageStatics& DamageStatics()
{
    static FDamageStatics Statics;
    return Statics;
}

UMyDamageExecution::UMyDamageExecution()
{
    RelevantAttributesToCapture.Add(DamageStatics().ArmorDef);
    RelevantAttributesToCapture.Add(DamageStatics().AttackPowerDef);
}
```

### Source (.cpp) - Execution Logic
```cpp
void UMyDamageExecution::Execute_Implementation(const FGameplayEffectCustomExecutionParameters& ExecutionParams, OUT FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const
{
    const FGameplayEffectSpec& Spec = ExecutionParams.GetOwningSpec();
    
    // Evaluation Parameters
    FAggregatorEvaluateParameters EvalParams;
    EvalParams.SourceTags = Spec.CapturedSourceTags.GetAggregatedTags();
    EvalParams.TargetTags = Spec.CapturedTargetTags.GetAggregatedTags();

    // 1. Capture Values
    float TargetArmor = 0.0f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(DamageStatics().ArmorDef, EvalParams, TargetArmor);

    float SourceAttack = 0.0f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(DamageStatics().AttackPowerDef, EvalParams, SourceAttack);

    // 2. Base Damage from SetByCaller (Optional)
    float BaseDamage = FMath::Max<float>(Spec.GetSetByCallerMagnitude(FGameplayTag::RequestGameplayTag(FName("Data.Damage")), false, 0.0f), 0.0f);

    // 3. The Formula
    float UnmitigatedDamage = BaseDamage + SourceAttack;
    float MitigatedDamage = FMath::Max<float>(UnmitigatedDamage - TargetArmor, 0.0f);

    // 4. Output the result to the "Damage" Meta Attribute
    if (MitigatedDamage > 0.0f)
    {
        OutExecutionOutput.AddOutputModifier(FGameplayModifierEvaluatedData(UMyAttributeSet::GetDamageAttribute(), EGameplayModOp::Additive, MitigatedDamage));
    }
}
```

## 3. Impact on Safety
- **Immutable Math**: ExecCalcs are run locally by the server (and predicted client). They guarantee the damage math cannot be tampered with by client-side blueprint hacks.
- **Snapshotting**: Capturing Source attributes with `Snapshot = true` ensures that if a fireball takes 5 seconds to travel, the damage uses the Attacker's power from the moment they cast it, even if their "Attack Buff" expired during flight.

## Verification Checklist
- [ ] Class inherits from `UGameplayEffectExecutionCalculation`.
- [ ] Attributes are correctly mapped to `Source` or `Target`.
- [ ] `AttemptCalculateCapturedAttributeMagnitude` is used to fetch values.
- [ ] Output modifies a Meta-Attribute (e.g., `Damage`) rather than modifying `Health` directly (allows `PostGameplayEffectExecute` to handle final application).
