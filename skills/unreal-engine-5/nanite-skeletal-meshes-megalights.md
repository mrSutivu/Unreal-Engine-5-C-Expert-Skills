---
name: nanite-skeletal-meshes-megalights
description: Optimizing for UE 5.5's newest features: Nanite Skeletal Meshes and MegaLights. Guidelines for implementation and bottleneck avoidance.
---

# Nanite Skeletal Meshes & MegaLights (UE5.5 Expert)

## Activation / When to Use
- Mandatory when targeting high-fidelity Next-Gen graphics.
- Trigger when dealing with massive crowds (Skeletal) or 1,000+ dynamic shadow-casting lights.

## 1. Nanite Skeletal Meshes (Experimental in 5.4, Refined in 5.5)
Allows cinematic-quality characters with millions of polygons to be rendered efficiently.

### Enablement
- Must be enabled per-mesh in the Skeletal Mesh Editor.
- C++ agents should check `SkeletalMesh->bEnableNanite`.

### Performance & Safety Constraints
- **HitproxyReadback Issue**: Currently, selecting massive Nanite Skeletal meshes in the Editor can cause extreme lag due to pixel-perfect hit proxy generation.
- **Deformation Overhead**: Nanite must update its clusters every frame based on bone transforms. Very high bone counts (1000+) can still bottleneck the CPU.
- **Materials**: Must use the Nanite path. WPO (World Position Offset) and Pixel Depth Offset are supported but expensive.

## 2. MegaLights (UE 5.5+)
A revolutionary system allowing scenes to have thousands of fully dynamic, shadow-casting lights without the traditional Deferred Rendering cost.

### Enablement
- Enabled via Project Settings or CVar: `r.MegaLights 1`.

### Best Practices
- **No More Light Limits**: You no longer need to strictly manage light attenuation radii to prevent overlap costs.
- **Volumetric Fog**: MegaLights currently have limitations or high costs when interacting with Volumetric Fog.
- **Shadow Proxies**: Works intimately with Virtual Shadow Maps (VSM) and Ray Traced Shadows. Ensure meshes have Nanite enabled to maximize MegaLight shadow performance.

## 3. Impact on Architecture
- Agents designing levels can freely place PointLights/SpotLights without needing complex C++ "Light Manager" pooling systems.
- Characters no longer require LODs (Level of Detail meshes), significantly reducing the asset management footprint for agents.

## Common Mistakes (BAD vs GOOD)

**BAD (Pre-5.5 Light Management)**:
```cpp
// Culling lights based on distance to save frames (Obsolete with MegaLights)
if (DistanceToPlayer > 5000) { MyLight->SetVisibility(false); }
```

**GOOD (UE 5.5 Workflow)**:
```text
Enable MegaLights. Allow the engine's BVH (Bounding Volume Hierarchy) to cull and shade thousands of lights natively on the GPU.
```

## Verification Checklist
- [ ] Skeletal Meshes targeted for crowds/cinematics have `bEnableNanite = true`.
- [ ] `r.MegaLights` is enabled for scenes with complex lighting.
- [ ] Translucent materials are minimized on Nanite Skeletal Meshes (still a performance trap).
