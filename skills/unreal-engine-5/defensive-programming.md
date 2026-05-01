---
name: defensive-programming
description: Expert guide for implementing rigorous nullptr checks and assertions (check, ensure, verify) in Unreal Engine 5.4. Distinguishes between fatal logic errors and recoverable runtime states to prevent data corruption.
---

# Defensive Programming (UE5 Expert)

## Activation / When to Use
- Mandatory for all pointer access and array indexing.
- Use when implementing core engine logic or critical gameplay systems.
- Trigger when handling "unexpected" states that indicate broken logic.
- DO NOT use fatal asserts (`check`) for user-input or recoverable network lag.

## 1. The Assertion Family (Expert Usage)

### check() / checkf() - Fatal Logic Errors
- **Behavior**: Immediate crash in Development/Editor. Stripped in Shipping (zero cost).
- **When to Use**: For "Impossible" states. If this fails, the code is fundamentally broken.
- **Example**: Mandatory component in `BeginPlay`.
```cpp
void AMyActor::BeginPlay() {
    Super::BeginPlay();
    checkf(MeshComp, TEXT("Actor %s missing mandatory MeshComponent!"), *GetName());
}
```

### ensure() / ensureAlways() - Non-Fatal Errors
- **Behavior**: Logs error + Callstack + Sends crash report. Execution CONTINUES. Returns `false` on failure.
- **When to Use**: For "Should Not Happen" states that can be handled gracefully (e.g., return early).
- **ensure**: Reports only ONCE per session.
- **ensureAlways**: Reports EVERY TIME it fails.
```cpp
if (ensureMsgf(WeaponData != nullptr, TEXT("WeaponData is null on %s! Combat might fail."), *GetName())) {
    WeaponData->Fire();
}
```

### verify() / verifyf() - Fatal with Side Effects
- **Behavior**: Expression is EVALUATED in Shipping, but crash only occurs in Development.
- **When to Use**: When the condition must run (e.g., loading an asset) but you want to assert its success during development.
```cpp
verifyf((MyTexture = LoadObject<UTexture2D>(...)) != nullptr, TEXT("Failed to load critical UI texture!"));
```

## 2. Nullptr Management Strategy

### "Expected" vs "Unexpected" Nulls
- **Expected Null**: A character might not have a weapon equipped. Use `if (Weapon)`.
- **Unexpected Null**: A Subsystem that should always exist is missing. Use `check(Subsystem)`.

### Defensive Pattern: The Early Return
Avoid deep nesting. Check and escape.
```cpp
void UMySystem::Process(AActor* Target) {
    if (!ensure(Target)) return; // Handle unexpected null
    
    UCharacterMovementComponent* MoveComp = Target->FindComponentByClass<UCharacterMovementComponent>();
    if (!MoveComp) return; // Expected null (actor might not be a character)

    MoveComp->StopMovementImmediately();
}
```

## 3. Impact on Safety
- **Data Integrity**: Asserts prevent writing corrupted state to the project’s permanent database or asset files.
- **Fail Fast**: Catch bugs at the source rather than debugging silent failures 100 frames later.

## Common Mistakes (BAD vs GOOD)

**BAD (Side effect in check)**:
```cpp
check(MyValue++); // ERROR: MyValue will NOT increment in Shipping!
```

**GOOD (Side effect in verify)**:
```cpp
verify(MyValue++); // SAFE: Increments in all builds, asserts in Dev.
```

**BAD (Silent failure)**:
```cpp
Pointer->DoSomething(); // CRASH: No check, no log, hard to debug.
```

**GOOD (Protected access)**:
```cpp
if (ensure(Pointer)) {
    Pointer->DoSomething();
}
```

## Verification Checklist
- [ ] No logical side effects inside `check()`.
- [ ] `check()` used for unrecoverable logic errors.
- [ ] `ensure()` used for recoverable but incorrect states.
- [ ] Meaningful error messages used in `*f` variants (e.g., `checkf`).
- [ ] Early returns used to prevent "Arrow Code" (deep nesting).
