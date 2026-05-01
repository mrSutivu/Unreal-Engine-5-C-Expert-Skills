---
name: ue5-native-memory-patterns
description: Comprehensive C++ templates for TSharedPtr, TUniquePtr, and FMemStack memory optimization.
---

# Native UE5 Memory Patterns (Expert Reference)

## Activation / When to Use
- Trigger when building pure C++ code (Non-UObjects) requiring manual memory management.
- Serves as the "Memory Dump" for fast transient allocations and smart pointer lifecycles.

## 1. Smart Pointers Template

### Class Definition
```cpp
#pragma once
#include "CoreMinimal.h"

// Pure C++ class (NO UCLASS!)
class FMyNativeSystem
{
public:
    FMyNativeSystem(int32 InID) : SystemID(InID) {}
    ~FMyNativeSystem() { UE_LOG(LogTemp, Log, TEXT("System Destroyed")); }

    void Execute() { /* logic */ }

private:
    int32 SystemID;
};
```

### Shared Ownership (TSharedPtr / TSharedRef)
```cpp
class FSystemManager
{
public:
    void Initialize()
    {
        // 1. Allocate block and object simultaneously
        SystemPtr = MakeShared<FMyNativeSystem>(42);

        // 2. Convert to SharedRef to guarantee non-null execution
        ProcessSystem(SystemPtr.ToSharedRef());
    }

    // TSharedRef guarantees the pointer is valid. No if(Valid) needed.
    void ProcessSystem(TSharedRef<FMyNativeSystem> InSystemRef)
    {
        InSystemRef->Execute();
    }

private:
    // Reference counted. Destroys FMyNativeSystem when this manager dies (if no one else holds a copy).
    TSharedPtr<FMyNativeSystem> SystemPtr;
};
```

### Exclusive Ownership (TUniquePtr)
```cpp
class FExclusiveManager
{
public:
    void Initialize()
    {
        // 1. Allocate
        UniqueSystem = MakeUnique<FMyNativeSystem>(99);
    }

    void TransferOwnership()
    {
        // 2. TUniquePtr cannot be copied. It MUST be moved.
        TUniquePtr<FMyNativeSystem> NewOwner = MoveTemp(UniqueSystem);
        
        // UniqueSystem is now nullptr.
        // NewOwner will destroy the object when it goes out of scope.
    }

private:
    TUniquePtr<FMyNativeSystem> UniqueSystem;
};
```

## 2. FMemStack Transient Allocation Template

When executing a heavy algorithm (like A* pathfinding or voxel meshing), allocating thousands of temporary `FVector`s or `structs` using `new` or `TArray.Add` creates a massive CPU bottleneck. `FMemStack` solves this.

```cpp
#include "Misc/MemStack.h"

struct FVoxelData
{
    FVector Position;
    int32 Density;
};

void ProcessVoxelChunk(int32 VoxelCount)
{
    // 1. Place a Mark on the Stack. 
    // This remembers the current memory offset.
    FMemMark Mark(FMemStack::Get());

    // ---------------------------------------------------------
    // METHOD A: Raw Array Allocation (Fastest)
    // Allocates space for VoxelCount items instantly.
    FVoxelData* RawVoxelBuffer = new (FMemStack::Get()) FVoxelData[VoxelCount];

    for (int32 i = 0; i < VoxelCount; ++i)
    {
        RawVoxelBuffer[i].Position = FVector(i, i, i);
        RawVoxelBuffer[i].Density = 1;
    }


    // ---------------------------------------------------------
    // METHOD B: TArray with MemStack Allocator (Safer/Dynamic)
    // TArray behaves normally, but its underlying memory uses the MemStack instead of the Heap.
    TArray<FVoxelData, TMemStackAllocator<>> VoxelArray;
    VoxelArray.Reserve(VoxelCount); // Pre-allocate on the stack
    
    for (int32 i = 0; i < VoxelCount; ++i)
    {
        VoxelArray.Add(FVoxelData{ FVector(i,0,0), 1 });
    }

    // ---------------------------------------------------------
    // 3. Auto-Cleanup
    // As soon as this function ends, the 'Mark' object goes out of scope.
    // Its destructor resets the MemStack offset to where it was at Step 1.
    // BOTH 'RawVoxelBuffer' and 'VoxelArray' are freed in 0 milliseconds.
}
```

### Safety Warning
If you return a pointer to memory allocated within the `FMemMark` scope, the engine will crash on the very next frame when it tries to read the overwritten stack memory.
```cpp
// FATAL ERROR
FVoxelData* GetVoxel() {
    FMemMark Mark(FMemStack::Get());
    FVoxelData* Data = new (FMemStack::Get()) FVoxelData();
    return Data; // Data becomes garbage immediately!
}
```
