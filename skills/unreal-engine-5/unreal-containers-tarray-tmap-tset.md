---
name: unreal-containers-tarray-tmap-tset
description: Selection logic and best practices for UE5 containers (TArray, TMap, TSet). Optimizes memory allocation, hashing, and retrieval.
---

# TArray, TMap, TSet Selection Logic (UE5 Expert)

## Activation / When to Use
- Mandatory when storing collections of data.
- Trigger when choosing data structures for inventories, registries, or spatial hashing.

## 1. TArray (The Workhorse)
- **Use Case**: Simple ordered lists, fast iteration, contiguous memory.
- **Performance**: Best for cache locality (fast CPU access). Slow for `Contains()` or `Remove()` on large datasets (O(N)).
- **Rule**: Default choice for 90% of tasks unless unique constraints or fast lookups are required.

```cpp
UPROPERTY()
TArray<TObjectPtr<AActor>> TargetList;

// Optimization: Pre-allocate memory if size is known
TargetList.Reserve(100);
```

## 2. TSet (Unique Hashing)
- **Use Case**: Storing UNIQUE items, fast membership testing.
- **Performance**: `Contains()`, `Add()`, and `Remove()` are O(1). Iteration is slower than `TArray`.
- **Rule**: Use when you need to answer "Has this item been processed?" repeatedly.

```cpp
UPROPERTY()
TSet<TObjectPtr<AActor>> ProcessedTargets;

if (!ProcessedTargets.Contains(NewTarget)) {
    ProcessedTargets.Add(NewTarget);
}
```

## 3. TMap (Key-Value Pairs)
- **Use Case**: Dictionaries, registries, linking IDs to data.
- **Performance**: O(1) lookups via Key.
- **Constraint**: Keys must implement `GetTypeHash` and `operator==`.

```cpp
UPROPERTY()
TMap<FName, FWeaponStats> WeaponRegistry;

if (FWeaponStats* Stats = WeaponRegistry.Find(WeaponID)) {
    // Modify Stats directly
}
```

## 4. Impact on Safety & Performance
- **Memory Thrashing**: Constantly adding/removing from a large `TArray` fragments memory. Use `TSet` or `TMap`.
- **Dangling Pointers**: As with all UE containers, if storing `UObjects`, the container MUST be marked `UPROPERTY()` to prevent GC issues.
- **Find vs Array[]**: Using `TMap::Find()` returns a pointer (safe null check). Using `TMap[]` asserts and crashes if the key doesn't exist.

## Common Mistakes (BAD vs GOOD)

**BAD (O(N) Search in Loop)**:
```cpp
// Large array
for (AActor* Target : AllActors) {
    if (BlacklistArray.Contains(Target)) continue; // VERY SLOW
}
```

**GOOD (O(1) Hash Lookup)**:
```cpp
// TSet Blacklist
for (AActor* Target : AllActors) {
    if (BlacklistSet.Contains(Target)) continue; // FAST
}
```

**BAD (Unsafe Map Access)**:
```cpp
FWeaponStats Stats = WeaponRegistry[WeaponID]; // CRASHES if WeaponID is not in map
```

**GOOD (Safe Map Access)**:
```cpp
if (FWeaponStats* Stats = WeaponRegistry.Find(WeaponID)) { ... }
```

## Verification Checklist
- [ ] `TArray` used for ordered iteration.
- [ ] `TSet` used for uniqueness and fast `Contains()`.
- [ ] `TMap` used for Key-Value lookup.
- [ ] Containers holding `UObject` pointers are marked `UPROPERTY()`.
- [ ] `Reserve()` used when container size is predictable.
