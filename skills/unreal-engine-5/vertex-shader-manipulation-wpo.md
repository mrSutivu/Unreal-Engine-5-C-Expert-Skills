---
name: vertex-shader-manipulation-wpo
description: Using World Position Offset (WPO) for procedural animations. Offloads animation logic from CPU to GPU.
---

# Vertex Shader Manipulation (WPO) (UE5 Expert)

## Activation / When to Use
- Mandatory for foliage wind, ocean waves, or simple ambient animations (hovering, pulsing).
- Trigger when you need to animate thousands of objects without destroying CPU performance.

## 1. Core Principles
World Position Offset (WPO) is a material pin that allows the vertex shader to move vertices on the GPU *before* they are rendered. The CPU (and C++) is completely unaware of this movement.

## 2. Implementation Rules

### Nanite Compatibility
In UE 5.4+, Nanite fully supports WPO, but it has a cost.
- **The Cost**: Nanite must re-evaluate cluster bounds if vertices move too far.
- **The Fix**: You MUST set the **"Max World Position Offset Displacement"** in the actor's details panel or C++. If vertices move beyond this box, the mesh will visually glitch or cull incorrectly.

```cpp
// In C++
MyStaticMeshComponent->SetMaxWorldPositionOffsetDisplacement(100.0f); // Prevents culling glitches
```

### The Math (Time-Based)
Use the `Time` node combined with `Sine` or `Cosine` multiplied by the `VertexNormalWS` or a specific vector.

```text
// Concept
WPO = sin(Time * Speed) * Amplitude * Z_Axis;
```

## 3. Impact on Safety & C++ Interaction
- **Physics Disconnect**: WPO is purely visual. collision bodies (Capsules, Boxes) DO NOT MOVE with WPO.
- **Gameplay Implication**: Never use WPO for an object the player must stand on or physically interact with, unless you also manually update the C++ collision to match the math.
- **Distance Culling**: Always fade out WPO over distance using the `PixelDepth` node to save GPU cycles on distant trees.

## Common Mistakes (BAD vs GOOD)

**BAD (CPU Animation for Crowds)**:
```cpp
void ATree::Tick(float DeltaTime) {
    // Moving 10,000 trees on the CPU will drop FPS to 5.
    AddActorLocalOffset(FVector(sin(Time), 0, 0));
}
```

**GOOD (GPU WPO)**:
```text
Disable Tick. Add Sine wave logic to the Tree's Material WPO pin. 10,000 trees run at 120 FPS.
```

**BAD (Unbounded Nanite WPO)**:
```text
Using heavy WPO on a Nanite mesh without setting Max Displacement. Shadows will tear and meshes will pop.
```

## Verification Checklist
- [ ] Procedural ambient animations are handled in the Material, not C++ `Tick`.
- [ ] `SetMaxWorldPositionOffsetDisplacement` is configured for Nanite meshes using WPO.
- [ ] Physics interactions are not expected to align perfectly with WPO bounds.
