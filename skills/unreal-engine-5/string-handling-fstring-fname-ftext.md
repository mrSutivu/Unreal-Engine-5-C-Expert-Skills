---
name: string-handling-fstring-fname-ftext
description: Strict string standards for Unreal Engine. Differentiates FString (manipulation), FName (fast lookup), and FText (localization).
---

# FString vs. FName vs. FText (UE5 Expert)

## Activation / When to Use
- Mandatory whenever declaring text-based variables or parameters.
- Essential for memory optimization and UI localization.

## 1. FString (Manipulation)
- **Use Case**: Dynamic text manipulation, concatenations, parsing, JSON generation.
- **Performance**: Heavy. Allocates memory dynamically. Slow comparisons.
- **Rule**: Use ONLY when the string changes at runtime.

```cpp
FString FullName = FString::Printf(TEXT("%s_%d"), *BaseName, ID);
FullName.Append(TEXT("_Processed"));
```

## 2. FName (Fast Lookup & IDs)
- **Use Case**: Asset paths, Bone names, Socket names, Tag identifiers.
- **Performance**: Ultra-light. Stored in a global string table. Comparisons are fast index checks (O(1)). Immutable (cannot be changed once created).
- **Rule**: Default choice for internal engine logic and identifiers.

```cpp
UPROPERTY(EditAnywhere)
FName MuzzleSocketName = FName("Muzzle_01");

// Very fast comparison
if (SocketName == MuzzleSocketName) { ... }
```

## 3. FText (User-Facing & Localization)
- **Use Case**: ANY text displayed to the player (UI, Subtitles, Tooltips).
- **Performance**: Heavier than FName. Contains formatting and localization keys.
- **Rule**: NEVER display an `FString` to a player. Always use `FText` to prevent technical debt during translation.

```cpp
// Define translatable text
FText WelcomeMsg = NSLOCTEXT("MyNamespace", "WelcomeKey", "Welcome to the game");

// Format translatable text
FText FormattedMsg = FText::Format(NSLOCTEXT("MyNamespace", "HPKey", "HP: {0}"), Health);
```

## 4. Conversions
Converting between types is common but carries overhead.
- `FName` -> `FString`: `MyName.ToString()`
- `FString` -> `FName`: `FName(*MyString)`
- `FString` -> `FText`: `FText::FromString(MyString)` (Avoid if possible, bypasses localization).
- `FText` -> `FString`: `MyText.ToString()`

## 5. Impact on Safety & Oversight
- **Localization Tech Debt**: Failing to use `FText` early means rewriting the entire UI layer later.
- **Memory Leaks**: Creating thousands of unique `FName`s dynamically (e.g., `FName(GuidString)`) can bloat the global name table, as `FName`s are never garbage collected. Use `FString` for highly volatile dynamic IDs.

## Common Mistakes (BAD vs GOOD)

**BAD (Slow comparison)**:
```cpp
FString Socket = "Muzzle";
if (Socket == "Muzzle") { ... } // Performs char-by-char string compare
```

**GOOD (Fast comparison)**:
```cpp
FName Socket = "Muzzle";
if (Socket == NAME_Muzzle) { ... } // Performs fast integer comparison
```

**BAD (Unlocalized UI)**:
```cpp
UTextBlock* NameText;
NameText->SetText(FText::FromString("Player 1")); // "Player 1" cannot be translated
```

**GOOD (Localized UI)**:
```cpp
NameText->SetText(LOCTEXT("PlayerNameKey", "Player 1"));
```

## Verification Checklist
- [ ] `FName` used for internal IDs, sockets, and asset paths.
- [ ] `FString` used for dynamic string manipulation.
- [ ] `FText` used for ALL UI elements and player-facing text.
- [ ] `NSLOCTEXT` or `LOCTEXT` macros used when initializing `FText`.
