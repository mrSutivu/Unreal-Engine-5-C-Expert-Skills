---
name: cpp20-lambda-captures
description: Guidelines for explicit lambda captures in C++20 for UE 5.4+. Ensures safety and clarity by avoiding implicit [=] capture of 'this'.
---

# C++20 Lambda Captures (UE5 Expert)

## Activation / When to Use
- Mandatory for all lambdas in UE 5.4+ (C++20 projects).
- Essential for async tasks, timers, and delegate bindings.
- Trigger when capturing class members or calling `this->Functions()`.

## 1. The C++20 Deprecation
In C++20, implicit capture of `this` via `[=]` is deprecated. It creates ambiguity because `[=]` suggests a copy of the object, while it actually copies the **pointer** to the object.

## 2. Recommended Syntax

### Explicit This Capture (Default)
Always explicitly capture `this` when accessing members.
```cpp
// DEPRECATED (Warning in 5.4)
auto BadLambda = [=]() { MyMember = 10; };

// CORRECT (UE5 standard)
auto GoodLambda = [=, this]() { MyMember = 10; };

// BEST (Selective capture)
auto BestLambda = [this, &CapturedVar]() { MyMember = CapturedVar; };
```

### Async Safety: Capture by Value (`[*this]`)
When a lambda outlives the object (e.g., `AsyncTask`), capture a **full copy** of the object to avoid "zombie" pointers.
```cpp
// SAFE for async: captures a copy of the object
auto AsyncLambda = [*this]() { 
    // Uses copy of members, safe even if original object is destroyed
    ProcessData(MyCopyOfData); 
};
```

## 3. Impact on Safety
- **Zombie Lambdas**: Prevents crash when a lambda executes after the object it references has been garbage collected.
- **Scope Clarity**: Forces the developer (or agent) to acknowledge exactly what is being stored in the lambda's closure.

## Common Mistakes (BAD vs GOOD)

**BAD (Unsafe Async Capture)**:
```cpp
FTimerHandle Handle;
GetWorld()->GetTimerManager().SetTimer(Handle, [this]() {
    DoSomething(); // CRASH if Actor is destroyed before timer fires!
}, 5.f, false);
```

**GOOD (Safe Delegate Binding)**:
```cpp
// Using TWeakObjectPtr for safety
TWeakObjectPtr<AMyActor> WeakThis(this);
GetWorld()->GetTimerManager().SetTimer(Handle, [WeakThis]() {
    if (WeakThis.IsValid()) {
        WeakThis->DoSomething();
    }
}, 5.f, false);
```

## Verification Checklist
- [ ] No `[=]` used alone (replace with `[=, this]` if needed).
- [ ] Async lambdas use `TWeakObjectPtr` or `[*this]` for safety.
- [ ] All captured variables are listed explicitly where possible for clarity.
