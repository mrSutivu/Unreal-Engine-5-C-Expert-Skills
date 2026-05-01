---
name: low-level-memory-fmemstack
description: High-performance temporary memory allocation using FMemStack and FMemMark. Bypasses standard malloc for lightning-fast transient arrays.
---

# Low-Level Memory Allocators (UE5 Expert)

## Activation / When to Use
- Mandatory when you need to allocate millions of temporary variables (Arrays, Matrices) inside a single function (e.g., Procedural Generation, complex AI pathfinding steps).
- Trigger when standard `TArray` reallocations are bottlenecking the CPU during a tight loop.

## 1. Core Principles: FMemStack
Calling `new` or adding elements to a `TArray` calls `malloc` on the OS, which is slow. Unreal Engine has pre-allocated memory stacks for the Game Thread and Render Thread. 
You can use `FMemStack` to instantly grab memory, and `FMemMark` to instantly free it all at once when the function ends.

## 2. Implementation Syntax

### Using the MemStack
```cpp
#include "Misc/MemStack.h"

void AMySystem::CalculateComplexPath()
{
    // 1. Place a Mark. This saves the current position of the stack.
    FMemMark Mark(FMemStack::Get());

    // 2. Allocate memory instantly from the stack (no malloc!)
    // e.g., We need space for 10,000 vectors temporarily
    int32 ArraySize = 10000;
    FVector* TempVectors = new (FMemStack::Get()) FVector[ArraySize];

    // 3. Do heavy math
    for (int32 i = 0; i < ArraySize; ++i)
    {
        TempVectors[i] = FVector(i, 0, 0); // Fast write
    }

    // 4. The magic: When the function ends, 'Mark' goes out of scope.
    // Its destructor instantly rewinds the stack, freeing all 10,000 vectors in 0 milliseconds.
}
```

### TArray on the Stack
You can force a `TArray` to use a specialized allocator so it doesn't use the heap.

```cpp
void AMySystem::ProcessLocally()
{
    // TMemStackAllocator forces the array to use FMemStack::Get()
    TArray<FVector, TMemStackAllocator<>> FastArray;
    
    // No heap allocation cost
    FastArray.Reserve(5000); 
    
    // Automatically cleaned up when function scope ends
}
```

### FMemory_Alloca
For very small, fast allocations, you can use the stack frame directly via `FMemory_Alloca` (maps to `_alloca`).
```cpp
void AMySystem::QuickBuffer()
{
    // Allocates 1024 bytes directly on the CPU stack. Extremely fast.
    uint8* Buffer = (uint8*)FMemory_Alloca(1024);
}
```

## 3. Impact on Safety
- **Scope Escaping**: Memory allocated on the `FMemStack` or via `_alloca` ceases to exist the moment the function returns. If you return a pointer to this memory or store it in a class variable, you will cause a catastrophic Access Violation crash.
- **Stack Overflow**: `FMemory_Alloca` is limited by the OS thread stack size (usually 1MB-2MB). Never allocate massive arrays with `_alloca`. Use `FMemStack` instead, which is backed by large engine-managed buffers.

## Common Mistakes (BAD vs GOOD)

**BAD (Returning Stack Memory)**:
```cpp
FVector* GetTempData() {
    FMemMark Mark(FMemStack::Get());
    FVector* Data = new (FMemStack::Get()) FVector();
    return Data; // FATAL: Data is destroyed when Mark goes out of scope!
}
```

**GOOD (Scope Confinement)**:
```cpp
void ProcessData() {
    FMemMark Mark(FMemStack::Get());
    TArray<FVector, TMemStackAllocator<>> LocalData;
    // Data is created, used, and destroyed safely within this bracket.
}
```

## Verification Checklist
- [ ] `FMemMark` is declared BEFORE any allocations on the `FMemStack`.
- [ ] Memory allocated this way NEVER leaves the scope of the function.
- [ ] `TMemStackAllocator` is used for transient arrays in hot loops.
- [ ] `FMemory_Alloca` is restricted to small allocations (< 100 KB).
