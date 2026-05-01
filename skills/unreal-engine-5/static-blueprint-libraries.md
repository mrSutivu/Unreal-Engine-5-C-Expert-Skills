---
name: static-blueprint-libraries
description: Exposing C++ logic via UBlueprintFunctionLibrary. Bridges the gap between native performance and EditorUtilityBlueprint flexibility.
---

# Static Blueprint Libraries (UE5 Expert)

## Activation / When to Use
- Mandatory when exposing pure math, generic utilities, or complex C++ algorithms to Blueprints.
- Use to create custom nodes for `EditorUtilityBlueprints` or Gameplay graphs.
- DO NOT use for stateful logic (Libraries cannot hold instance variables safely).

## 1. Core Principles
- **Stateless**: All functions must be `static`. No class instances are required to call them.
- **Global Access**: Nodes are available anywhere in any Blueprint.

## 2. Implementation Syntax

### Header (.h)
```cpp
#pragma once
#include "CoreMinimal.h"
#include "Kismet/BlueprintFunctionLibrary.h"
#include "MyUtilsLibrary.generated.h"

UCLASS()
class MYPROJECT_API UMyUtilsLibrary : public UBlueprintFunctionLibrary
{
    GENERATED_BODY()

public:
    // Pure function (Green node, no execution pins)
    UFUNCTION(BlueprintPure, Category = "MyProject|Math")
    static float CalculateComplexTrajectory(FVector Start, FVector End);

    // Executable function (Blue node, has execution pins)
    UFUNCTION(BlueprintCallable, Category = "MyProject|System")
    static void ForceGarbageCollection();
};
```

### Source (.cpp)
```cpp
float UMyUtilsLibrary::CalculateComplexTrajectory(FVector Start, FVector End)
{
    // Heavy C++ math here
    return FVector::Distance(Start, End) * 3.14f;
}

void UMyUtilsLibrary::ForceGarbageCollection()
{
    GEngine->ForceGarbageCollection(true);
}
```

## 3. World Context Object
If your static function needs to spawn an actor or access the `UWorld`, you must provide a WorldContextObject.

```cpp
// Header
UFUNCTION(BlueprintCallable, meta = (WorldContext = "WorldContextObject"), Category = "MyProject")
static void SpawnSystem(const UObject* WorldContextObject);

// CPP
void UMyUtilsLibrary::SpawnSystem(const UObject* WorldContextObject)
{
    if (UWorld* World = GEngine->GetWorldFromContextObject(WorldContextObject, EGetWorldErrorMode::LogAndReturnNull))
    {
        // Safe to use World
    }
}
```

## 4. Impact on Safety
- **Performance**: Keeps Editor tools and Gameplay logic fast by ensuring tight loops run in native C++.
- **Reusability**: Write once in C++, use infinitely in UI, Editor Tools, and Game logic.

## Verification Checklist
- [ ] Class inherits from `UBlueprintFunctionLibrary`.
- [ ] All `UFUNCTION` methods are `static`.
- [ ] `BlueprintPure` used for getter/math functions (no side effects).
- [ ] `BlueprintCallable` used for functions that modify state.
- [ ] `WorldContext` meta tag used when `GetWorld()` is required.
