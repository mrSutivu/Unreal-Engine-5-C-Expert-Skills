---
name: ui-performance-invalidation-boxes
description: Utilizing Invalidation Boxes to cache widget painting and avoid using Tick for UI updates. Critical for 120+ FPS UMG performance.
---

# UI Performance & Invalidation Boxes (UE5 Expert)

## Activation / When to Use
- Mandatory when building complex UIs with hundreds of elements (e.g., Inventories, Skill Trees, Minimaps).
- Trigger when the UI is causing frame drops (visible via `stat statgroup_slate`).
- NEVER use `Tick` for UI property updates.

## 1. The UMG Performance Bottleneck
By default, Unreal evaluates and paints every visible widget every frame. If a widget uses "Property Binding" (e.g., binding HealthText to a `GetHealth()` function), that function is called every frame, destroying CPU performance.

## 2. Event-Driven UI (The Solution)
Never use UMG Property Bindings. Use **Delegates/Events** to push data to the UI only when it changes.

```cpp
// BAD (Called 60+ times a second)
float UMyWidget::GetHealthPercent() { return Player->Health / Player->MaxHealth; }

// GOOD (Called ONLY when damaged)
void UMyWidget::OnDamageTaken(float NewHealth) { HealthBar->SetPercent(NewHealth / MaxHealth); }
```

## 3. Invalidation Boxes
An `InvalidationBox` is a special UMG container. It caches the visual representation (geometry and paint commands) of all its children. The engine will NOT redraw the children unless explicitly told they are "Invalidated."

### Implementation in UMG
1. Wrap your complex static UI (e.g., an action bar) in an `Invalidation Box`.
2. Check `Cache Relative Transforms` if the widgets don't move.

### Manual Invalidation (C++)
If data inside the box changes, you must invalidate it to force a redraw.

```cpp
#include "Components/InvalidationBox.h"

void UMyUIWidget::UpdateStaticText()
{
    MyStaticText->SetText(FText::FromString("New Data"));
    
    // Force the Invalidation Box to redraw its cached contents on the next frame
    if (MyInvalidationBox)
    {
        MyInvalidationBox->InvalidateCache();
    }
}
```

## 4. Retainer Boxes (Advanced)
A `RetainerBox` is similar but renders its children to an off-screen texture (Render Target).
- **Use Case**: Applying post-process materials to UI (e.g., CRT distortion) or deliberately reducing the UI framerate (e.g., rendering a minimap at 15fps while the game runs at 120fps).

## 5. Impact on Safety
- **CPU Starvation**: Prevents the Game Thread from being bogged down by Slate Draw operations.
- **Scalability**: Ensures the agent can generate massive UI tools for the Editor without crashing the user's workspace.

## Common Mistakes (BAD vs GOOD)

**BAD (Tick-based UI)**:
```cpp
void UMyUIWidget::NativeTick(const FGeometry& MyGeometry, float InDeltaTime) {
    PlayerNameText->SetText(Player->GetName()); // Evaluates every frame!
}
```

**GOOD (Event-based + Invalidation)**:
```cpp
void UMyUIWidget::NativeConstruct() {
    Player->OnNameChanged.AddDynamic(this, &UMyUIWidget::UpdateName);
}
```

## Verification Checklist
- [ ] No Property Bindings are used in the UMG designer (use variables and set them manually).
- [ ] `NativeTick` is not overridden in `UUserWidget` unless absolutely necessary for animation.
- [ ] Static UI blocks are wrapped in an `InvalidationBox`.
- [ ] Dynamic updates trigger `InvalidateCache()` if inside a cached box.
