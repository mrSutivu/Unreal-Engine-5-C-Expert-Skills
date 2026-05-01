---
name: shader-parameter-structs
description: Using BEGIN_SHADER_PARAMETER_STRUCT to bind C++ data to HLSL cleanly via RDG. Eliminates legacy RHI boilerplates.
---

# Shader Parameter Structs (UE5 Expert)

## Activation / When to Use
- Mandatory for passing data (Floats, Vectors, Textures, Buffers) from C++ to Global Shaders in UE5.
- Trigger when updating legacy UE4 code to the new Render Dependency Graph (RDG) system.

## 1. The RDG Revolution
In UE4, binding parameters required hundreds of lines of `LAYOUT_FIELD` and manual `SetShaderValue` calls. In UE5, the `BEGIN_SHADER_PARAMETER_STRUCT` macros automatically generate the reflection data needed by the Render Dependency Graph (RDG).

## 2. Implementation Syntax

### The C++ Struct
Define this inside your shader class or standalone.

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FMyShaderParameters, )
    // Simple Variables (Must match HLSL types exactly)
    SHADER_PARAMETER(FVector4f, TintColor)
    SHADER_PARAMETER(float, Intensity)

    // Textures & Samplers
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, InputTexture)
    SHADER_PARAMETER_SAMPLER(SamplerState, InputSampler)

    // Compute Shader UAV (Read/Write buffers)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<float4>, OutputTexture)

    // Render Targets (For Pixel Shaders)
    RENDER_TARGET_BINDING_SLOTS()
END_SHADER_PARAMETER_STRUCT()
```

### The HLSL Equivalent
The variable names in HLSL MUST match the C++ struct exactly.

```hlsl
// In MyShader.usf
float4 TintColor;
float Intensity;

Texture2D InputTexture;
SamplerState InputSampler;

RWTexture2D<float4> OutputTexture;
```

## 3. Passing the Struct via RDG
When adding the pass, you allocate and fill the struct.

```cpp
FMyShaderParameters* PassParameters = GraphBuilder.AllocParameters<FMyShaderParameters>();
PassParameters->TintColor = FVector4f(1, 0, 0, 1);
PassParameters->Intensity = 5.0f;
// Bind RDG resources...
PassParameters->InputTexture = MyRDGTexture;
```

## 4. Impact on Safety
- **Type Safety**: The macros ensure C++ types align perfectly with HLSL expectations.
- **RDG Validation**: RDG uses these structs to understand resource dependencies (e.g., knowing that `OutputTexture` will be written to), preventing race conditions on the GPU.

## Common Mistakes (BAD vs GOOD)

**BAD (Legacy UE4 Binding)**:
```cpp
// Writing custom Serialize and Bind functions in C++ is obsolete and incompatible with modern RDG tracking.
```

**GOOD (RDG Struct)**:
```cpp
// Using SHADER_PARAMETER_* macros allows RDG to automatically track resource lifecycles.
```

## Verification Checklist
- [ ] `BEGIN_SHADER_PARAMETER_STRUCT` is used for all shader inputs/outputs.
- [ ] C++ types map correctly to HLSL (`FVector4f` -> `float4`, `float` -> `float`).
- [ ] `RDG` specific macros (`SHADER_PARAMETER_RDG_TEXTURE`) are used for resources managed by `FRDGBuilder`.
