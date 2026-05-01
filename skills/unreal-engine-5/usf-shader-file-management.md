---
name: usf-shader-file-management
description: Best practices for managing .usf and .ush files. Covers virtual shader directories and include paths in Unreal Engine 5.
---

# USF Shader File Management (UE5 Expert)

## Activation / When to Use
- Mandatory when writing custom HLSL shaders for Unreal Engine.
- Trigger when organizing a plugin or project that contains native shader code.

## 1. USF vs USH
- **`.usf` (Unreal Shader File)**: Contains the main entry point for a shader (e.g., `MainPS` for Pixel Shader, `MainCS` for Compute Shader).
- **`.ush` (Unreal Shader Header)**: Contains reusable HLSL functions, macros, or structs. Included via `#include`.

## 2. Virtual Shader Directories
Unreal uses a virtual file system to map physical folders to shader paths.
- **Engine Shaders**: Mapped to `/Engine/`.
- **Project Shaders**: Mapped to `/Project/`. (Requires enabling "Allow Project Shaders" in Project Settings).
- **Plugin Shaders**: Mapped to `/Plugin/YourPluginName/`.

## 3. C++ Registration (Plugins)
To allow Unreal to find your plugin's shaders, you MUST register the virtual path in your module's `StartupModule`.

```cpp
#include "Interfaces/IPluginManager.h"
#include "Misc/Paths.h"
#include "ShaderCore.h"

void FMyShaderPluginModule::StartupModule()
{
    FString PluginShaderDir = FPaths::Combine(IPluginManager::Get().FindPlugin(TEXT("MyShaderPlugin"))->GetBaseDir(), TEXT("Shaders"));
    AddShaderSourceDirectoryMapping(TEXT("/Plugin/MyShaderPlugin"), PluginShaderDir);
}
```

## 4. Include Syntax in HLSL
Always use absolute virtual paths to prevent compilation errors when shaders are included from different directories.

```hlsl
// BAD (Relative paths can break)
#include "../Common/Math.ush"

// GOOD (Virtual absolute path)
#include "/Plugin/MyShaderPlugin/Common/Math.ush"

// Including Engine headers
#include "/Engine/Public/Platform.ush"
```

## 5. Impact on Safety
- **Cross-Platform Compilation**: Proper `.ush` separation ensures that standard Engine HLSL (which handles platform-specific quirks like Vulkan vs DX12) can be included cleanly.
- **Hot Reloading**: Proper virtual mapping allows the Engine to track file changes and recompile shaders live when you press `Ctrl+Shift+.`.

## Verification Checklist
- [ ] Shader entry points use `.usf`, helper functions use `.ush`.
- [ ] Virtual directories (`AddShaderSourceDirectoryMapping`) are registered in `StartupModule`.
- [ ] `#include` directives use absolute virtual paths (e.g., `/Plugin/...`).
