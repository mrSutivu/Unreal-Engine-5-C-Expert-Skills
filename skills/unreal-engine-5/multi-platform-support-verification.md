---
name: multi-platform-support-verification
description: Validating builds for cross-platform support (Windows, Linux, Consoles). Ensures C++ code is platform-agnostic for Fab verification.
---

# Multi-Platform Support Verification (UE5 Expert)

## Activation / When to Use
- Mandatory when submitting C++ plugins to Fab Marketplace.
- Trigger when using system-level APIs (File I/O, threading, rendering).

## 1. The Fab Platform Mandate
Fab's automated bots compile your C++ plugin against multiple platforms (typically Windows, Linux, and Mac). If your code uses Windows-specific headers (e.g., `#include <windows.h>`), it will fail Linux validation and be rejected.

## 2. Platform-Agnostic Abstractions (HAL)
Unreal Engine provides the Hardware Abstraction Layer (HAL) to replace OS-specific C++ libraries. You MUST use these.

### File Operations
- **BAD**: `std::ifstream`, `fopen`.
- **GOOD**: `FFileHelper`, `IFileManager`.
```cpp
FString FileContent;
FFileHelper::LoadFileToString(FileContent, *FilePath);
```

### Threading
- **BAD**: `std::thread`, `pthread`.
- **GOOD**: `FRunnable`, `AsyncTask`, `FGraphEvent`.
```cpp
AsyncTask(ENamedThreads::AnyBackgroundThreadNormalTask, []() { ... });
```

### Time and System
- **BAD**: `std::chrono`, `GetTickCount()`.
- **GOOD**: `FPlatformTime::Seconds()`, `FDateTime`.

## 3. Whitelisting Platforms (.uplugin)
If your plugin relies on a third-party `.dll` that only exists on Windows, you must restrict the plugin from being compiled on other platforms to avoid Fab rejection.

```json
"Modules": [
    {
        "Name": "MyWindowsOnlyModule",
        "Type": "Runtime",
        "LoadingPhase": "Default",
        "PlatformAllowList": [ "Win64" ]
    }
]
```

## 4. Conditional Compilation
If you need platform-specific logic, use the UE preprocessor macros.

```cpp
#if PLATFORM_WINDOWS
    // Windows specific SDK code
#elif PLATFORM_LINUX
    // Linux equivalent
#else
    UE_LOG(LogTemp, Error, TEXT("Platform not supported."));
#endif
```

## 5. Impact on Safety
- **Store Acceptance**: 90% of C++ plugin rejections on the Marketplace are due to Linux/Mac compilation failures caused by `std::` libraries or missing includes.
- **Cross-Platform Stability**: Relying on UE's HAL ensures memory allocation and threading behave consistently across all consoles and PC.

## Verification Checklist
- [ ] No standard OS headers (`windows.h`, `unistd.h`) are included globally.
- [ ] UE HAL (`FFileHelper`, `FPlatformTime`, `FGenericPlatformMath`) replaces `std::` equivalents.
- [ ] Platforms lacking required third-party binaries are excluded via `PlatformAllowList`.
- [ ] Code builds successfully using the Unreal Automation Tool (UAT) `BuildPlugin` command.
