---
name: tonemapper-pipeline-architecture
description: Understanding the UE5 Tonemapping pipeline. Crucial for advanced color grading, ACES integration, and HDR rendering.
---

# Tonemapper Pipeline Architecture (UE5 Expert)

## Activation / When to Use
- Mandatory when modifying how colors translate from HDR linear space to LDR screen space.
- Trigger when implementing custom camera film emulation or overriding the ACES approximation.

## 1. The Core Purpose
Unreal Engine calculates lighting in **HDR Linear Space** (values can go infinitely high, e.g., 100.0 for the sun). Monitors display in **LDR (0 to 1)**. The Tonemapper compresses the infinite HDR range into the 0-1 range gracefully, mimicking physical film.

## 2. The File Architecture
The Tonemapping logic is split across three primary files in `Engine/Shaders/Private/`.

### `TonemapCommon.ush`
Contains the pure mathematical functions.
- `FilmToneMap()`: The core function. In UE5, this approximates the Academy Color Encoding System (ACES) filmic response.
- Exposes parameters like Toe, Slope, Shoulder, and WhiteClip.

### `PostProcessCombineLUTs.usf`
The Color Grading baker.
- Unreal does NOT execute the heavy ACES math on every pixel during gameplay.
- Instead, this shader runs once (when Post Process settings change) to bake the Tonemapping curve AND all Color Grading (Color Wheels, Saturation, Contrast) into a **3D LUT (Look-Up Table)**.

### `PostProcessTonemap.usf`
The final pixel shader applied to the screen.
- Reads the HDR `SceneColor`.
- Looks up the baked value in the 3D LUT (`ColorLookupTable` function).
- Applies Eye Adaptation (Auto Exposure) and Dithering to prevent color banding.

## 3. Architectural Rule: Where to Modify?
- **To change the math (e.g., replace ACES with AgX)**: Modify `TonemapCommon.ush`. This ensures Color Grading still works and is baked into the LUT correctly.
- **To add an effect AFTER Tonemapping (e.g., UI Overlay)**: Modify `PostProcessTonemap.usf` or use an `ISceneViewExtension` injected `AfterTonemap`.
- **To add an effect BEFORE Tonemapping (e.g., Chromatic Aberration)**: Use a standard Post Process Material set to `Before Tonemapping`.

## 4. Impact on Safety & Performance
- **LUT Performance**: Understanding that Tonemapping is baked into a LUT is critical. Forcing the math to run per-pixel in `PostProcessTonemap.usf` destroys rendering performance.
- **Color Integrity**: Outputting values > 1.0 after the Tonemapper will cause severe visual artifacts (blooming) on standard monitors unless targeting specific HDR displays.

## Verification Checklist
- [ ] Custom Tonemapper math is implemented in `TonemapCommon.ush` or `PostProcessCombineLUTs.usf`.
- [ ] `PostProcessTonemap.usf` is reserved for final color application and dithering.
- [ ] Modifications respect the 3D LUT generation to preserve Color Grading functionality.
