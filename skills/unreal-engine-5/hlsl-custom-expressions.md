---
name: hlsl-custom-expressions
description: Writing raw HLSL shader code within the Material Editor via Custom Nodes. Handles complex mathematical effects safely.
---

# HLSL Custom Expressions (UE5 Expert)

## Activation / When to Use
- Use when an effect requires loops (`for`, `while`), complex math (Raymarching, Fractals), or custom texture sampling logic that is inefficient in visual nodes.
- Trigger when standard material graphs become impossibly large or difficult to maintain.

## 1. Core Principles
The `Custom` node in the Material Editor allows raw HLSL code. In UE5, this code is injected directly into the generated shader during compilation.

## 2. Best Practices for HLSL in UE5

### 1. The Virtual File Pattern
Instead of writing 500 lines of HLSL directly in the tiny Custom Node text box, create a `.usf` (Unreal Shader File) or `.ush` file in your project's `Shaders/` folder.

**In the Custom Node:**
```hlsl
#include "/Project/MyShaders/ComplexMath.ush"
return CalculateFractal(UV, Iterations);
```

*Note: Requires enabling "Allow Project Shaders" in Project Settings.*

### 2. Handling Inputs
Define exact inputs in the Custom Node properties. If you define `float3 A` and `float B`, your code can use them directly.

```hlsl
// Example: Simple Custom Node Code
float result = 0;
for(int i=0; i < Iterations; i++) {
    result += A.x * B;
}
return result;
```

## 3. Impact on Safety & Performance
- **Cross-Platform Compilation**: HLSL written in Custom Nodes must compile to GLSL (Vulkan), Metal, and console shading languages via Unreal's Shader Cross-Compiler. Avoid highly platform-specific intrinsic functions.
- **Optimization Restrictions**: The UE Material Compiler cannot heavily optimize or fold code inside a Custom Node. Only use them when absolutely necessary; visual nodes are generally faster for simple math due to aggressive compiler folding.

## Common Mistakes (BAD vs GOOD)

**BAD (Simple Math in HLSL)**:
```hlsl
return A * B + C; // BAD: Defeats the material compiler's ability to optimize. Use Add/Multiply nodes.
```

**GOOD (Required HLSL)**:
```hlsl
// Raymarching loop (Impossible with standard nodes)
float dist = 0;
for(int i=0; i<64; i++) {
    dist += GetSceneDistance(Pos + Dir * dist);
}
return dist;
```

## Verification Checklist
- [ ] HLSL code does not replicate logic easily done with standard nodes.
- [ ] Code is stored in `.ush` files for complex algorithms to allow IDE syntax highlighting.
- [ ] Inputs and Output Types are strictly defined in the node properties.
