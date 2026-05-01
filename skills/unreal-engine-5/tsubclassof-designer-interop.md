---
name: tsubclassof-designer-interop
description: Expert usage of TSubclassOf<T> to manage Blueprint class assignments in C++. Ensures type safety for spawning and dynamic class selection.
---

# TSubclassOf for Designer Interop (UE5 Expert)

## Activation / When to Use
- Mandatory when a variable needs to store a **class type** (blueprint) rather than an instance.
- Use for Spawning logic (Projectiles, Enemies, Effects).
- Trigger when designers need to pick a class in the Editor.

## 1. Why TSubclassOf?
A raw `UClass*` allows assigning ANY class in the engine, which is unsafe. `TSubclassOf<T>` constrains the selection to `T` and its children.

- **Editor UI**: Limits the dropdown to compatible classes.
- **Type Safety**: Prevents runtime crashes when casting or spawning.

## 2. Implementation Syntax

### Declaration in Header
```cpp
UPROPERTY(EditAnywhere, Category = "Combat")
TSubclassOf<AActor> ProjectileClass;
```

### Usage in Source
```cpp
if (ProjectileClass) {
    GetWorld()->SpawnActor<AActor>(ProjectileClass, SpawnLocation, SpawnRotation);
}
```

## 3. Advanced Filtering
You can use `TSubclassOf` with Interfaces to ensure a designer picks a class that implements a specific functionality.
```cpp
UPROPERTY(EditAnywhere, Category = "Combat")
TSubclassOf<UInterface_Interactable> InteractableClass;
```

## 4. Impact on Safety
- **Functional Constraints**: Prevents a designer from assigning a "UserInterface" class to a "Projectile" variable, which would crash the engine upon instantiation.
- **Contract Enforcement**: Ensures the agent creates dynamic systems that remain within a "Safe" range defined by the architecture.

## Common Mistakes (BAD vs GOOD)

**BAD (Unsafe UClass)**:
```cpp
UPROPERTY(EditAnywhere)
UClass* EnemyClass; // Designer can pick "Texture2D" or "SoundWave" - CRASH on spawn.
```

**GOOD (Constrained Selection)**:
```cpp
UPROPERTY(EditAnywhere)
TSubclassOf<APawn> EnemyClass; // Only Pawns or children appear in dropdown.
```

## Pro Tips
- **Abstract Classes**: Use `meta = (AllowAbstract = "false")` to prevent designers from picking classes that cannot be instantiated.
- **Null Checks**: Always check `if (MyClass)` before calling `SpawnActor`.

## Verification Checklist
- [ ] `TSubclassOf<T>` used instead of `UClass*` for class references.
- [ ] Template type `T` is as specific as possible (e.g., `TSubclassOf<ACharacter>` vs `TSubclassOf<AActor>`).
- [ ] Variables are checked for validity before spawning.
- [ ] Categories are assigned for easy designer access.
