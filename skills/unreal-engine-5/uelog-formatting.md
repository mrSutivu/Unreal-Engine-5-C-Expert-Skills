---
name: uelog-formatting
description: Mastery of structured logging in UE 5.4+ using UE_LOG and UE_LOGFMT. Guidelines for verbosity levels, formatting, and safety.
---

# UE_LOG Formatting and Verbosity (UE5 Expert)

## Activation / When to Use
- Mandatory for all diagnostic output.
- Use for error reporting, state tracking, and performance monitoring.
- Trigger when implementing new systems or debugging existing ones.

## 1. UE_LOGFMT (The 5.4+ Standard)
UE 5.4+ strongly prefers `UE_LOGFMT` (Structured Logging) for performance and type safety.
- **Header**: `#include "Logging/StructuredLog.h"`
- **Benefits**: No `TEXT()` required for format, no `*` needed for `FString`, type-checked at compile time.

```cpp
// Modern Style (UE_LOGFMT)
UE_LOGFMT(LogTemp, Log, "Player {Name} reached level {Level}", ("Name", PlayerName), ("Level", CurrentLevel));

// Positional Style
UE_LOGFMT(LogTemp, Log, "Damage: {0} from {1}", DamageAmount, DamageSource);
```

## 2. UE_LOG (Classic Style)
Use for legacy maintenance or simple quick logs.
- **Formatting**: Requires `*` to dereference `FString` into `TCHAR*`.
```cpp
// Classic Style
UE_LOG(LogTemp, Warning, TEXT("Actor %s is overlapping %s"), *GetName(), *OtherActor->GetName());
```

## 3. Verbosity Levels
Choose the correct level to avoid log spam while ensuring critical errors are seen.

| Level | Usage | Visible in Console? |
| :--- | :--- | :--- |
| **Fatal** | Critical error. Crashes the engine. | Yes |
| **Error** | Significant failure. Highlighted in RED. | Yes |
| **Warning** | Potential issue. Highlighted in YELLOW. | Yes |
| **Display** | High-level info. | Yes |
| **Log** | Standard info. | No (Editor Log only) |
| **Verbose** | Detailed debug info. | No |

## 4. Custom Log Categories
Avoid using `LogTemp` for permanent systems.
```cpp
// In .h
DECLARE_LOG_CATEGORY_EXTERN(LogMyProject, Log, All);

// In .cpp
DEFINE_LOG_CATEGORY(LogMyProject);
```

## 5. Impact on Safety
- **Oversight Capacity**: Clear, structured logs allow the user (and the agent) to diagnose failures instantly without deep debugging.
- **Silent Failure Prevention**: Warnings and Errors provide a trail when logic deviates from the plan.

## Common Mistakes (BAD vs GOOD)

**BAD (Missing dereference in UE_LOG)**:
```cpp
UE_LOG(LogTemp, Log, TEXT("Name is %s"), MyString); // CRASH or Garbage Output
```

**GOOD (Correct dereference)**:
```cpp
UE_LOG(LogTemp, Log, TEXT("Name is %s"), *MyString);
```

**BAD (Spamming LogTemp)**:
```cpp
UE_LOG(LogTemp, Log, TEXT("Tick")); // Impossible to filter in a large project.
```

**GOOD (Structured & Categorized)**:
```cpp
UE_LOGFMT(LogCombat, Verbose, "Actor {0} Ticked", GetName());
```

## Verification Checklist
- [ ] `UE_LOGFMT` used for new code (with `StructuredLog.h`).
- [ ] Correct verbosity level selected (Warning/Error for failures).
- [ ] `FStrings` are dereferenced with `*` in classic `UE_LOG`.
- [ ] No permanent debug logs in `Tick` or frequent loops.
- [ ] Custom log categories used for distinct systems.
