---
name: uproperty-specifiers
description: Comprehensive guide for UPROPERTY specifiers in UE 5.4. Detailed visibility matrix, metadata tricks (EditCondition, Widgets), networking (Replication), and serialization (Config, Transient).
---

# UPROPERTY Specifiers (UE5 Expert)

## Activation / When to Use
- Use for all class members requiring reflection, serialization, or Editor visibility.
- Mandatory for variables accessed by Blueprints.
- Essential for networking and persistence.

## 1. Visibility & Editing Matrix
| Specifier | Editor Archetype (CDO) | Level Instance | Usage |
| :--- | :--- | :--- | :--- |
| `EditAnywhere` | Read/Write | Read/Write | Generic properties. |
| `EditDefaultsOnly` | Read/Write | Hidden | Base config (Stats, Meshes). |
| `EditInstanceOnly` | Hidden | Read/Write | Unique per-instance data. |
| `VisibleAnywhere` | Read-only | Read-only | Debug data, Components pointers. |
| `VisibleDefaultsOnly` | Read-only | Hidden | Archetype-only debug info. |
| `VisibleInstanceOnly` | Hidden | Read-only | Runtime state tracking. |

## 2. Blueprint Interaction
- `BlueprintReadOnly`: Get node only.
- `BlueprintReadWrite`: Get & Set nodes.
- `BlueprintAssignable`: For `multicast dynamic delegates` (Events).
- `BlueprintCallable`: For `dynamic delegates` (Functions).
- `Category = "Main\|Sub"`: Organizes details panel. Use pipe `\|` for nesting.

## 3. Serialization & Logic Flags
- `Config`: Read/Write from `.ini` files. Class must have `Config=Game` in `UCLASS`.
- `Transient`: NOT saved to disk. Reset to default on load. Use for runtime caches.
- `DuplicateTransient`: Reset to default when duplicating Actor (Copy/Paste).
- `SaveGame`: Included in `USaveGame` automated serialization systems.
- `Instanced`: For object pointers. Creates unique instance of the object for each parent instance. Essential for complex data structures.

## 4. Advanced Metadata (`meta`)
Metadata adds logic to the Editor UI without affecting runtime.

- `DisplayName = "Custom Name"`: Overrides variable name in UI.
- `ToolTip = "Help text"`: Overrides C++ comments for hover help.
- `ClampMin / ClampMax`: Forces hard limits on numeric entry.
- `UIMin / UIMax`: Limits slider range but allows manual entry outside.
- `EditCondition = "bMyBoolean"`: Grays out property if boolean is false.
- `EditConditionHides`: Hides property completely if `EditCondition` is false.
- `AllowPrivateAccess = "true"`: Exposes `private` member to Blueprints.
- `MakeEditWidget`: (FVector/FRotator) Adds interactive gizmo in Viewport.
- `Categories = "Tag.Category"`: Limits `FGameplayTag` selection to a sub-branch.

## 5. Components & Subobjects Rule
CRITICAL: Pointers to `CreateDefaultSubobject` MUST be `VisibleAnywhere`.
**WHY**: `EditAnywhere` allows deleting/replacing the pointer, breaking the Actor hierarchy. `VisibleAnywhere` locks the pointer but allows editing the component's internal properties.

```cpp
// GOOD
UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
TObjectPtr<UStaticMeshComponent> MeshComp;

// BAD
UPROPERTY(EditAnywhere, Category = "Components")
TObjectPtr<UStaticMeshComponent> MeshComp;
```

## 6. Networking & Replication
- `Replicated`: Standard sync across network.
- `ReplicatedUsing = OnRep_Function`: Calls function on Clients when value arrives.
- `NotReplicated`: Explicitly skips sync (default).

## 7. UE 5.4 Special Features
- `FieldNotify`: Used with MVVM and CommonUI to notify UI of property changes.
- `Interp`: Mark for Sequencer/Matinee tracks.
- `AssetRegistrySearchable`: Properties searchable in Content Browser without loading assets.

## Common Mistakes (BAD vs GOOD)

**BAD (Specifier Conflict)**:
```cpp
UPROPERTY(EditAnywhere, VisibleAnywhere) // ERROR: Mutually exclusive
float Speed;
```

**BAD (Missing Category)**:
```cpp
UPROPERTY(EditAnywhere) // Clutters "None" category in UI
int32 Health;
```

**GOOD (Clean & Safe)**:
```cpp
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Combat", meta = (ClampMin = "0.0", UIMin = "0.0"))
float BaseDamage;
```

## Verification Checklist
- [ ] No `Edit*` and `Visible*` mixed on same property.
- [ ] Component pointers are `VisibleAnywhere`.
- [ ] Categories are assigned and hierarchical where possible.
- [ ] `meta` used for UX (Clamps, EditConditions).
- [ ] `BlueprintReadOnly` used for properties that should not be set by designers.
- [ ] `Transient` used for data that shouldn't persist in levels/saves.
