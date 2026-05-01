---
name: smart-reference-counting
description: Mastery of UPROPERTY() for Garbage Collector (GC) tracking. Ensures strong reference chains for UObjects to prevent Use-After-Free errors and premature deallocation.
---

# Smart Reference Counting (UE5 Expert)

## Activation / When to Use
- Mandatory for all `UObject` pointers held as class members.
- Use when spawning temporary actors or creating dynamic data objects.
- Essential for long-running workflows to maintain memory integrity.

## 1. The UPROPERTY() Mandate
The Unreal Garbage Collector (GC) ONLY tracks pointers marked with `UPROPERTY()`.
- **Reference Collection**: During the "Scan" phase, GC follows `UPROPERTY` paths.
- **Reachability**: Any object not reachable from a "Root" object (e.g., World, GameInstance) or a strong `UPROPERTY` chain is collected.

```cpp
// GC PROTECTED - Object will live as long as AMyActor lives
UPROPERTY()
TObjectPtr<UUserWidget> MyWidget;

// GC IGNORED - Pointer will point to garbage memory after 60s (Default GC cycle)
UUserWidget* VulnerableWidget; 
```

## 2. Maintaining the Strong Reference Chain
An object must be linked to a persistent Root.
1. **Actor Components**: Automatically referenced by the Actor.
2. **Subobjects**: Use `CreateDefaultSubobject` or mark as `Instanced` + `UPROPERTY`.
3. **Dynamic Objects**: Must be stored in a `TArray`, `TMap`, or a member variable marked `UPROPERTY()`.

## 3. Advanced Reference Handling

### Root Set (AddReferencedObjects)
If you must store `UObjects` in a non-UClass (native C++), you MUST implement `FGCObject` and override `AddReferencedObjects`.
```cpp
class FMyNativeClass : public FGCObject {
    UObject* MyObj;
    virtual void AddReferencedObjects(FReferenceCollector& Collector) override {
        Collector.AddReferencedObject(MyObj);
    }
};
```

### Write Barriers (TObjectPtr)
In UE 5.4+, `TObjectPtr` acts as a **Write Barrier**. Assigning a value notifies the Incremental GC immediately, improving performance and reliability.

## 4. Impact on Safety
- **Use-After-Free Prevention**: Prevents accessing memory that has been reclaimed.
- **Oversight Stability**: Critical for agents spawning "Research Actors". If the agent loses the reference chain, the research data is wiped by GC.

## Common Mistakes (BAD vs GOOD)

**BAD (Dangling Pointer)**:
```cpp
void UMySystem::SpawnData() {
    MyData = NewObject<UMyData>(this); // If MyData is NOT a UPROPERTY, it's garbage soon.
}
```

**GOOD (Strong Reference)**:
```cpp
UPROPERTY()
TObjectPtr<UMyData> PersistentData;

void UMySystem::SpawnData() {
    PersistentData = NewObject<UMyData>(this); // GC now tracks this reference.
}
```

**BAD (Naked TArray)**:
```cpp
TArray<UObject*> ObjectList; // Pointers inside are NOT tracked by GC.
```

**GOOD (Tracked Container)**:
```cpp
UPROPERTY()
TArray<TObjectPtr<UObject>> ObjectList; // GC tracks every element.
```

## Pro Tips
- **Weak Pointers**: Use `TWeakObjectPtr<T>` if you need to reference an object WITHOUT preventing its destruction.
- **Soft Pointers**: Use `TSoftObjectPtr<T>` for lazy-loaded assets (prevents immediate memory bloat).

## Verification Checklist
- [ ] Every `UObject*` (or `TObjectPtr`) member variable is marked `UPROPERTY()`.
- [ ] Containers (`TArray`, `TMap`) containing `UObjects` are marked `UPROPERTY()`.
- [ ] Native C++ classes holding `UObjects` implement `FGCObject`.
- [ ] No raw `UObject*` used for long-term storage in non-reflected classes.
