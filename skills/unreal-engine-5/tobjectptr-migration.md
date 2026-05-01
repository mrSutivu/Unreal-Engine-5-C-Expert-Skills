---
name: tobjectptr-migration
description: Expert guide for migrating raw pointers to TObjectPtr in Unreal Engine 5. Covers lazy loading, editor stability, performance impacts, and edge cases for UE 5.4.
---

# TObjectPtr Migration (UE5 Expert)

## Activation / When to Use
- Use for all `UObject` pointer members in class headers (`.h`).
- Use when preparing project for Asset Virtualization.
- Mandatory for new UE5 project standards.
- DO NOT use for: local variables, function parameters, return types, or non-UObject types.

## Core Principles
- **Lazy Loading**: `TObjectPtr` enables Editor to defer object loading until accessed via `->`.
- **Access Tracking**: Tracks object usage in Editor. Improves stability during asset deletion/reloads.
- **Zero Cost Shipped**: Compiles to raw `T*` in non-editor builds. No overhead in production.
- **Operator Overloading**: Mimics raw pointer syntax (`->`, `*`, `==`, `bool`).

## Implementation: Header vs Source

### Header (.h)
ALWAYS use `TObjectPtr` for `UPROPERTY` members.
```cpp
UPROPERTY(EditAnywhere, Category = "Combat")
TObjectPtr<USoundBase> FireSound;

UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
TObjectPtr<UStaticMeshComponent> Mesh;
```

### Source (.cpp)
USE raw pointers for transients and logic. `TObjectPtr` converts implicitly.
```cpp
void AMyActor::PlaySound()
{
    // Implicit conversion to raw pointer
    USoundBase* Sound = FireSound; 
    if (Sound)
    {
        UGameplayStatics::PlaySoundAtLocation(this, Sound, GetActorLocation());
    }
}
```

## Edge Cases & Advanced Handling

### 1. Pointer References (`T*&`)
`TObjectPtr<T>` is NOT `T*`. Passing it to a function expecting `T*&` fails.
**FIX**: Use `.GetPtrRef()` or temporary variable.
```cpp
// Function signature: void ModifyPointer(UObject*& OutPtr);
UObject* TempPtr = MyTObjectPtr.Get();
ModifyPointer(TempPtr);
MyTObjectPtr = TempPtr; // Re-assign
```

### 2. Casting
Standard `Cast<T>` works directly on `TObjectPtr`.
```cpp
TObjectPtr<AActor> MyActor;
UCharacterMovementComponent* MoveComp = Cast<UCharacterMovementComponent>(MyActor);
```

### 3. Non-UObjects
`TObjectPtr` ONLY supports `UObject` derived classes. 
- `FVector*` -> Stay raw.
- `IInterface*` -> Use `TScriptInterface<IInterface>` or raw pointer depending on context.

### 4. Container Types
- `TArray<TObjectPtr<T>>` -> Correct for class members.
- `TMap<K, TObjectPtr<V>>` -> Correct for class members.

## Common Mistakes (BAD vs GOOD)

**BAD (Local variable overhead)**:
```cpp
void Process() {
    TObjectPtr<AActor> LocalActor = GetActor(); // Extra tracking overhead in Editor
}
```
**GOOD (Lightweight transients)**:
```cpp
void Process() {
    AActor* LocalActor = GetActor(); 
}
```

**BAD (Parameter Bloat)**:
```cpp
void UpdateMesh(TObjectPtr<UStaticMesh> NewMesh); // Forces caller to use TObjectPtr
```
**GOOD (Flexible API)**:
```cpp
void UpdateMesh(UStaticMesh* NewMesh); // Accepts TObjectPtr or Raw*
```

## Pro Tips
- **Otool**: Use `UnrealObjectPtrTool` for mass migration of legacy UE4 codebases.
- **Debugging**: In Editor, use `MyPtr.Get()` if you need to inspect the raw address in some specific debugger views.
- **Safety**: `TObjectPtr` handles `nullptr` exactly like raw pointers.

## Verification Checklist
- [ ] All `UPROPERTY` pointers in `.h` use `TObjectPtr<T>`.
- [ ] No `TObjectPtr` used as function parameters or return types.
- [ ] No `TObjectPtr` used as local variables.
- [ ] Raw pointer references (`*&`) handled via `.Get()` or temp variables.
- [ ] Project compiles in both DebugGame Editor and Shipping configurations.
