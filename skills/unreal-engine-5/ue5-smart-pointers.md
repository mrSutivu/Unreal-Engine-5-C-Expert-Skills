---
name: ue5-smart-pointers
description: Utilizing TSharedPtr, TSharedRef, and TUniquePtr for native C++ memory management. Contrasts with UObjects and std:: libraries.
---

# UE5 Smart Pointers (UE5 Expert)

## Activation / When to Use
- Mandatory when managing the lifecycle of pure C++ classes or Structs that DO NOT inherit from `UObject`.
- Trigger when building native Editor UIs (Slate), custom network packets, or raw multithreading payloads.
- **NEVER** use with `UObject` or `AActor`.

## 1. The Unreal Standard vs STD
Do not use `std::shared_ptr`. Unreal uses its own allocators (`FMemory`) to track memory leaks and optimize pools.
- `TSharedPtr<T>`: A reference-counted pointer. Can be `nullptr`.
- `TSharedRef<T>`: A reference-counted pointer that is **GUARANTEED** to never be null.
- `TUniquePtr<T>`: Exclusive ownership. Cannot be copied, only moved.

## 2. Implementation Syntax

### TUniquePtr (Strict Ownership)
Use for systems that have a clear owner.

```cpp
class FMyNativeSystem { ... };

// Header
TUniquePtr<FMyNativeSystem> MySystem;

// Source
MySystem = MakeUnique<FMyNativeSystem>(); // Allocates
// MySystem is automatically destroyed when the owning class is destroyed.
```

### TSharedPtr & TSharedRef (Shared Ownership)
Use `MakeShared<T>()` to allocate. It is vastly more performant than `MakeShareable(new T())` because it allocates the control block and the object in a single heap operation.

```cpp
class FMySharedData { ... };

// 1. Creation
TSharedPtr<FMySharedData> DataPtr = MakeShared<FMySharedData>();

// 2. Passing as a Guarantee (TSharedRef)
// If a function requires data, ask for a TSharedRef. You avoid writing `if(Ptr.IsValid())`.
void ProcessData(TSharedRef<FMySharedData> ValidData) {
    ValidData->DoWork(); 
}

// 3. Converting Ptr to Ref
if (DataPtr.IsValid()) {
    ProcessData(DataPtr.ToSharedRef());
}
```

## 3. Thread Safety
By default, `TSharedPtr` is **NOT** thread-safe to increment/decrement its reference count (for maximum performance). If you pass a `TSharedPtr` to an async background thread, you MUST declare it as thread-safe.

```cpp
// Thread-safe declaration
TSharedPtr<FMySharedData, ESPMode::ThreadSafe> SafeData = MakeShared<FMySharedData, ESPMode::ThreadSafe>();
```

## 4. Impact on Safety
- **The UObject Trap**: Wrapping a `UObject*` in a `TSharedPtr` will cause a fatal crash. The Smart Pointer will try to `delete` the object when the ref count hits 0, but the Garbage Collector also tries to delete it. Use `TObjectPtr` or `TWeakObjectPtr` for `UObjects`.
- **Cyclic References**: If `TSharedPtr A` holds `TSharedPtr B`, and B holds A, they never die. Use `TWeakPtr` for the backward link to break the cycle.

## Common Mistakes (BAD vs GOOD)

**BAD (Standard Library)**:
```cpp
std::shared_ptr<FMyData> Data = std::make_shared<FMyData>(); // Violates UE5 tracking.
```

**BAD (UObject wrapping)**:
```cpp
TSharedPtr<AActor> MyActor; // FATAL: Double-delete collision with Garbage Collector.
```

**GOOD (Native Classes Only)**:
```cpp
TSharedPtr<FMyNativeStruct> Data = MakeShared<FMyNativeStruct>(); // SAFE
```

## Verification Checklist
- [ ] `TSharedPtr` / `TUniquePtr` are used EXCLUSIVELY for non-`UObject` types.
- [ ] `MakeShared<T>()` is used instead of `MakeShareable(new T())`.
- [ ] `TSharedRef` is used in function parameters to guarantee non-null execution.
- [ ] `std::` pointers are entirely avoided.
