---
name: performance-struct-padding-cache
description: Optimizing CPU Cache Locality via Struct Padding. Ordering variables from largest to smallest to prevent memory bloat.
---

# Struct Padding & Cache Locality (UE5 Expert)

## Activation / When to Use
- Mandatory when defining new `USTRUCT()` or `UCLASS()` data blocks that will be spawned hundreds or thousands of times.
- Trigger when memory usage is high or array iteration is slow.

## 1. The Compiler Padding Problem
C++ compilers align variables in memory for fast CPU access. If you mix large and small variables randomly, the compiler inserts "Padding" (empty, wasted bytes) between them.

**BAD STRUCTURE (Wasted Memory)**:
```cpp
USTRUCT()
struct FBadStats
{
    GENERATED_BODY()
    
    bool bIsDead;        // 1 byte
    // Compiler adds 7 bytes of padding here!
    double BaseDamage;   // 8 bytes
    bool bIsStunned;     // 1 byte
    // Compiler adds 3 bytes of padding here!
    int32 MaxHealth;     // 4 bytes
};
// Total Size: 24 Bytes (10 bytes wasted!)
```

## 2. The Solution: Largest to Smallest
To eliminate padding and keep your structs perfectly packed, **ALWAYS order variables from Largest (Pointers/64-bit) to Smallest (Bools/8-bit).**

**GOOD STRUCTURE (Perfectly Packed)**:
```cpp
USTRUCT()
struct FGoodStats
{
    GENERATED_BODY()
    
    double BaseDamage;   // 8 bytes
    int32 MaxHealth;     // 4 bytes
    bool bIsDead;        // 1 byte
    bool bIsStunned;     // 1 byte
    // 2 bytes of padding at the end for alignment
};
// Total Size: 16 Bytes (0 internal waste!)
```

## 3. The Size Hierarchy Guide
Order your properties strictly in this format:

1. **Pointers** (`AActor*`, `TObjectPtr<T>`) -> *8 Bytes*
2. **64-bit Primitives** (`double`, `int64`) -> *8 Bytes*
3. **Complex Structs** (`FVector`, `FRotator`) -> *24 Bytes (Usually 3 doubles)*
4. **32-bit Primitives** (`float`, `int32`) -> *4 Bytes*
5. **16-bit Primitives** (`int16`, `uint16`) -> *2 Bytes*
6. **8-bit Primitives** (`int8`, `uint8`) -> *1 Byte*
7. **Booleans** (`bool`) -> *1 Byte*

## 4. Cache Locality (The True Goal)
Why does saving 8 bytes matter? Because CPUs read memory in "Cache Lines" (blocks of 64 bytes).
- If your struct is 64 bytes, the CPU reads exactly 1 struct per cycle.
- If your struct is bloated to 72 bytes due to bad padding, the CPU must read TWO cache lines to get one struct. When iterating over a `TArray` of 10,000 items, this padding destroys CPU performance due to "Cache Misses".

## 5. Boolean Bitfields (Ultimate Packing)
If you have 8 booleans, they take 8 bytes. You can compress them into 1 byte using Bitfields.

```cpp
// 8 Bools = 1 Byte total.
uint8 bIsDead : 1;
uint8 bIsStunned : 1;
uint8 bIsPoisoned : 1;
```

## Verification Checklist
- [ ] Variables in structs and classes are ordered from Largest byte size to Smallest byte size.
- [ ] Pointers (`8 bytes`) are always at the top.
- [ ] Booleans (`1 byte`) are always at the bottom.
- [ ] Bitfields (`uint8 bFlag : 1;`) are used when 3 or more booleans are present.
