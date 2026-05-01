---
name: declare-global-shader
description: Creating C++ Global Shaders using DECLARE_GLOBAL_SHADER and IMPLEMENT_GLOBAL_SHADER.
---

# Declare Global Shader (UE5 Expert)

## Activation / When to Use
- Mandatory when writing standalone Shaders (Compute Shaders, custom Post-Process passes) that do not belong to a Material.
- Trigger when bridging a `.usf` file to C++.

## 1. Core Principles
A Global Shader in Unreal is a C++ class that inherits from `FGlobalShader`. It tells the engine how to compile the HLSL code and how to bind parameters.

## 2. Implementation Syntax

### Header (.h)
Use `DECLARE_GLOBAL_SHADER` to generate the necessary reflection boilerplates.

```cpp
#pragma once
#include "GlobalShader.h"
#include "ShaderParameterStruct.h"

class MYPLUGIN_API FMyCustomPixelShader : public FGlobalShader
{
    DECLARE_GLOBAL_SHADER(FMyCustomPixelShader);
    SHADER_USE_PARAMETER_STRUCT(FMyCustomPixelShader, FGlobalShader);

    // See "Shader Parameter Structs" skill for this macro
    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER_TEXTURE(Texture2D, InputTexture)
        SHADER_PARAMETER_SAMPLER(SamplerState, InputSampler)
        RENDER_TARGET_BINDING_SLOTS()
    END_SHADER_PARAMETER_STRUCT()
};
```

### Source (.cpp)
Use `IMPLEMENT_GLOBAL_SHADER` to link the C++ class to the `.usf` file and specify the entry point.

```cpp
#include "MyCustomShader.h"

// Parameters: Class, Virtual Path, Entry Point Function, Shader Type (Pixel, Compute, Vertex)
IMPLEMENT_GLOBAL_SHADER(FMyCustomPixelShader, "/Plugin/MyShaderPlugin/MyPixelShader.usf", "MainPS", SF_Pixel);
```

## 3. Compilation Control (ShouldCompilePermutation)
You can prevent the shader from compiling on platforms that don't support it (e.g., preventing a heavy Compute Shader from compiling on Mobile).

```cpp
// In the Header
static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters)
{
    // Only compile for SM5 (DirectX 11/12, High-end Vulkan)
    return IsFeatureLevelSupported(Parameters.Platform, ERHIFeatureLevel::SM5);
}
```

## 4. Impact on Safety
- **Cook Times**: `ShouldCompilePermutation` prevents unnecessary shader compilations for unsupported platforms, drastically speeding up packaging.
- **Linker Errors**: Failing to use `IMPLEMENT_GLOBAL_SHADER` in exactly one `.cpp` file will cause LNK2019 errors.

## Verification Checklist
- [ ] Class inherits from `FGlobalShader`.
- [ ] `DECLARE_GLOBAL_SHADER` is in the class body.
- [ ] `SHADER_USE_PARAMETER_STRUCT` is used for modern parameter binding.
- [ ] `IMPLEMENT_GLOBAL_SHADER` connects the class to a valid `.usf` path and entry point.
- [ ] `ShouldCompilePermutation` checks for supported Feature Levels.
