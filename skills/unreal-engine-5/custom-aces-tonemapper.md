---
name: custom-aces-tonemapper
description: Modifying the ACES tonemapper in UE5. Demonstrates replacing FilmToneMap via Post Process Materials or engine shader overrides.
---

# Custom ACES Tonemapper (UE5 Expert)

## Activation / When to Use
- Trigger when a project requires a non-standard color response (e.g., AgX, Reinhard, or a custom cinematic look).
- Use to replace the default UE5 ACES filmic approximation.

## 1. The Mathematical Base (ACES Approximation)
The standard Stephen Hill / Narkowicz ACES fit used widely in realtime graphics:

```hlsl
float3 ACESFilm(float3 x) {
    float a = 2.51f;
    float b = 0.03f;
    float c = 2.43f;
    float d = 0.59f;
    float e = 0.14f;
    return saturate((x * (a * x + b)) / (x * (c * x + d) + e));
}
```

## 2. Method A: The Post Process Material (Non-Destructive)
Ideal for testing or project-specific overrides without modifying Engine source.

### Material Setup
1. Create a Material, Domain: `Post Process`.
2. Blendable Location: `Replacing the Tonemapper`.
3. Add a `Custom` node with the HLSL code.

### Node Configuration
- Input: `SceneTexture: PostProcessInput0` (This is the HDR Linear Scene Color).
- *Warning*: Replacing the tonemapper this way bypasses Unreal's Auto Exposure and Color Grading. You must implement exposure multiplication manually inside the material: `InputColor * Exposure`.

## 3. Method B: Modifying Engine Shaders (Production)
For deep integration that preserves the Post Process Volume's Color Grading sliders.

### Step 1: Open TonemapCommon.ush
Navigate to `Engine/Shaders/Private/TonemapCommon.ush`.

### Step 2: Override `FilmToneMap`
Find `half3 FilmToneMap( half3 LinearColor )` and replace its logic.

```hlsl
// Example: Replacing default math with a Custom Reinhard
half3 FilmToneMap( half3 LinearColor ) 
{
    // Custom Reinhard implementation
    return LinearColor / (LinearColor + half3(1.0, 1.0, 1.0));
}
```

### Step 3: Recompile Shaders
In the Unreal Editor, use the shortcut `Ctrl+Shift+.` to trigger a global shader recompile, immediately reflecting the new tonemapper across the entire engine and generating a new 3D LUT.

## 4. Impact on Safety
- **Engine Source Maintenance**: Modifying `TonemapCommon.ush` affects ALL projects using that Engine build. For studios, this requires a custom Engine Fork.
- **Color Clamping**: The custom function MUST return `saturate` (values between 0.0 and 1.0). Returning HDR values (> 1.0) into the LDR pipeline causes blown-out UI and screen tearing.

## Common Mistakes (BAD vs GOOD)

**BAD (Unclamped Return)**:
```hlsl
return LinearColor * 2.0f; // Exceeds 1.0. Causes artifacts.
```

**GOOD (Saturated Return)**:
```hlsl
return saturate(LinearColor * 2.0f); // Safe LDR output.
```

## Verification Checklist
- [ ] If using a Post Process Material, `Blendable Location` is exactly `Replacing the Tonemapper`.
- [ ] If modifying `.ush`, `Ctrl+Shift+.` is used to force a recompile.
- [ ] The custom math outputs values clamped between 0 and 1.
