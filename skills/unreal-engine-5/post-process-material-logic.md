---
name: post-process-material-logic
description: Developing shaders for final rendering passes. Implements screen-space effects like outlines and custom depth filtering.
---

# Post-Process Material Logic (UE5 Expert)

## Activation / When to Use
- Mandatory when creating screen-wide effects (Night Vision, Cel-Shading, Outline Highlights).
- Trigger when you need to manipulate the final rendered frame using G-Buffer data (Depth, Normals, Custom Stencil).

## 1. Core Principles
Post-Process Materials run after the main scene is rendered. They read from screen-space buffers and output a final color.

### Material Setup
- **Material Domain**: Post Process.
- **Output**: Emissive Color.
- **Blendable Location**: Typically `Before Tonemapping` (for lighting-accurate effects) or `After Tonemapping` (for UI-like overlays).

## 2. Reading the G-Buffer
Use the `SceneTexture` node to access render data.

- **SceneColor**: The actual rendered image.
- **SceneDepth**: Distance from camera to pixel.
- **CustomStencil**: An integer mask. Critical for rendering outlines only on specific objects.

## 3. C++ Integration (Custom Stencil)
To highlight an object (e.g., when the agent selects it), C++ must write to the Custom Stencil buffer.

### 1. Enable in Project Settings
`r.CustomDepth = 3` (Enabled with Stencil).

### 2. C++ Activation
```cpp
void AMyActor::Highlight()
{
    if (MeshComp)
    {
        MeshComp->SetRenderCustomDepth(true);
        MeshComp->SetCustomDepthStencilValue(255); // Match this ID in the PostProcess Material
    }
}
```

## 4. Impact on Safety & Performance
- **Heavy GPU Cost**: A post-process material evaluates for *every single pixel* on the screen. Complex math here drops framerates drastically on 4K monitors.
- **Resolution Scaling**: Ensure effects scale correctly. `SceneDepth` is in Unreal Units, but screen coordinates are UVs (0 to 1).

## Common Mistakes (BAD vs GOOD)

**BAD (Unbounded PP)**:
```text
Running a massive blur kernel on a Post Process material set to 'After Tonemapping' without downsampling. Massive GPU hit.
```

**GOOD (Optimized Outlines)**:
```text
Using `CustomStencil` to isolate outline calculations ONLY to pixels near the edges of stencil ID 255.
```

## Verification Checklist
- [ ] Material Domain is set to `Post Process`.
- [ ] `SceneTexture: PostProcessInput0` is used to pass through the original scene color if no effect is applied.
- [ ] C++ interaction uses `SetRenderCustomDepth` and `SetCustomDepthStencilValue`.
- [ ] Project settings have Custom Depth-Stencil Pass enabled.
