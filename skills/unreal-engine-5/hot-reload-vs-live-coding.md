---
name: hot-reload-vs-live-coding
description: Live Coding standards in UE 5.5. Understanding why Hot Reload is deprecated and how to safely update logic without crashing the Editor.
---

# Hot Reload vs. Live Coding (UE5 Expert)

## Activation / When to Use
- Mandatory when editing C++ code while the Unreal Editor is open.
- Trigger when deciding how to compile changes iteratively.

## 1. The Death of Hot Reload
**Hot Reload is DEPRECATED and DANGEROUS in UE5.**
- **Why it fails**: Hot Reload creates new versions of UClasses in memory and tries to patch existing instances (e.g., `HOTRELOADED_AMyActor_01`). This leads to blueprint corruption, variable resets, and massive crashes.
- **Rule**: NEVER use the "Compile" button in the Editor toolbar if Hot Reload is enabled, and NEVER build from Visual Studio/Rider while the Editor is open unless using Live Coding.

## 2. Live Coding (The Standard)
**Live Coding** is the industry standard for UE 5.4/5.5. It patches the executable in memory without creating duplicate UClasses.

### How to use:
1. Ensure Live Coding is enabled (Ctrl+Alt+F11 to compile).
2. Edit your `.cpp` files in your IDE.
3. Press `Ctrl+Alt+F11` (or click the Live Coding icon in the Editor).
4. The changes are patched instantly into the running Editor.

## 3. Live Coding Limitations (Header Changes)
Live Coding excels at modifying logic (`.cpp`), but has strict limitations regarding memory layout (`.h`).

### Safe to Live Code:
- Changing logic inside a `.cpp` function.
- Adding a new `UFUNCTION` or `UPROPERTY` (in UE 5.4+, Live Coding experimental features support this, but it requires Editor restart to see new variables in Blueprints).

### UNSAFE / Requires Editor Restart:
- Changing the size of a `USTRUCT`.
- Changing the base class of a `UCLASS`.
- Removing a `UPROPERTY` or changing its type.
- Modifying constructor initialization lists that affect memory layout.

## 4. Impact on Safety
- **Blueprint Corruption**: Disabling Hot Reload prevents the agent/user from permanently corrupting Blueprint assets that reference C++ classes.
- **Iteration Speed**: Live Coding allows instantaneous logic updates, maximizing developer productivity.

## Common Mistakes (BAD vs GOOD)

**BAD (Hot Reload Workflow)**:
```text
1. Edit C++ file.
2. Build from Visual Studio while Editor is open.
3. Editor attempts Hot Reload.
4. Blueprints referencing the class break or lose data.
```

**GOOD (Live Coding Workflow)**:
```text
1. Edit C++ file.
2. Press Ctrl+Alt+F11 in Editor or IDE.
3. Live Coding patches logic instantly.
```

## Verification Checklist
- [ ] Hot Reload is disabled in Editor Preferences.
- [ ] Live Coding is enabled.
- [ ] Editor is closed and restarted if major header changes (memory layout) are made.
