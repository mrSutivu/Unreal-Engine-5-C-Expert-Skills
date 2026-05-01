---
name: forward-declaration
description: Principles for reducing header inter-dependencies and compile times using forward declarations. Follows Epic's strict standards for module management.
---

# Forward Declaration (UE5 Expert)

## Activation / When to Use
- Mandatory for all class/struct pointer references in header files (`.h`).
- Trigger when adding a new member variable or function parameter.
- DO NOT use for: parent classes (inheritance), non-pointer members (by-value), or enum usage.

## 1. Core Principles
Forward declaring tells the compiler "this type exists" without loading its entire definition.
- **Reduces Compile Times**: Fewer files to re-parse when a header changes.
- **Breaks Circular Dependencies**: Allows Class A to reference Class B while Class B references Class A.
- **Cleaner Headers**: Headers only include what they absolutely need for compilation.

## 2. Implementation Syntax

### Using keywords in member declarations
Preferred for `UPROPERTY` pointers.
```cpp
UPROPERTY(EditAnywhere, Category = "Components")
class USpringArmComponent* CameraBoom;

UPROPERTY(EditAnywhere, Category = "Data")
struct FGameplayTagContainer ImpactTags;
```

### Top-of-file declarations
Preferred for clarity and multiple references.
```cpp
#pragma once
#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyActor.generated.h"

// Forward Declarations
class UCameraComponent;
class UNiagaraComponent;
struct FMyCustomData;

UCLASS()
class AMyActor : public AActor { ... };
```

## 3. Class vs Struct Rules
You MUST match the definition keyword to avoid "Mismatched Tag" errors (UHT/MSVC).
- **class**: For `UObject`, `AActor`, `UActorComponent`, and native classes.
- **struct**: For `USTRUCT()`, `FVector`, `FRotator`, and native structs.

| Prefix | Use | Example |
| :--- | :--- | :--- |
| **U** | `class` | `class UStaticMesh;` |
| **A** | `class` | `class APlayerController;` |
| **F** | `struct` | `struct FHitResult;` |
| **I** | `class` | `class IMyInterface;` |

## 4. Impact on Safety
- **Compilation Cascades**: Prevents a single change in a low-level header from triggering a 30-minute rebuild of the entire project.
- **Decoupling**: Prevents the agent from creating "Spaghetti Headers" where every file depends on every other file.

## Common Mistakes (BAD vs GOOD)

**BAD (Include Bloat)**:
```cpp
// MyActor.h
#include "Components/StaticMeshComponent.h"
#include "NiagaraComponent.h"
#include "PhysicsEngine/PhysicsSettings.h" // Unnecessary bloat!
```

**GOOD (Surgical Includes)**:
```cpp
// MyActor.h - Forward declare
class UStaticMeshComponent;
class UNiagaraComponent;

// MyActor.cpp - Include only what you USE
#include "Components/StaticMeshComponent.h"
#include "NiagaraComponent.h"
```

## Verification Checklist
- [ ] No unnecessary `#include` in `.h` (moved to `.cpp`).
- [ ] All pointer-only references in `.h` are forward declared.
- [ ] Keywords (`class`/`struct`) match the actual type definition.
- [ ] No forward declaration for inheritance (base classes MUST be included).
