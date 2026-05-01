---
name: visual-logger-vlog
description: Implementing UE_VLOG for advanced temporal debugging of AI and gameplay systems. Records states and 3D shapes over time.
---

# Visual Logger (UE_VLOG) (UE5 Expert)

## Activation / When to Use
- Mandatory for debugging AI (Behavior Trees, Pathfinding) or complex spatial algorithms.
- Trigger when `UE_LOG` text is insufficient to understand *where* something happened in the world.

## 1. Core Principles
The Visual Logger (Tools -> Debug -> Visual Logger) records a timeline of events. For every frame, it saves text, Actor state, and 3D shapes (spheres, lines) that you can scrub through backward and forward like a video.

## 2. Implementation Syntax

### Setup
Must include the Visual Logger header.
```cpp
#include "VisualLogger/VisualLogger.h"
```

### Logging Text & State
Logs text to a specific category, but attaches it to an Actor in the timeline.

```cpp
void AMyAIController::MakeDecision()
{
    // Vlogs the string to the "LogAI" category, associated with 'this' actor
    UE_VLOG(this, LogAI, Log, TEXT("AI decided to attack target %s"), *Target->GetName());
}
```

### Logging 3D Shapes
Draws shapes in the Visual Logger viewport (not the game viewport).

```cpp
void AMyAIController::CalculatePath()
{
    FVector Start = GetPawn()->GetActorLocation();
    FVector End = Start + FVector(500, 0, 0);

    // Draws an arrow in the VisLog showing the calculated path
    UE_VLOG_ARROW(this, LogAI, Log, Start, End, FColor::Red, TEXT("Calculated Path"));
    
    // Draws a sphere at the destination
    UE_VLOG_LOCATION(this, LogAI, Log, End, 50.0f, FColor::Green, TEXT("Destination"));
}
```

## 3. Impact on Safety & Debugging
- **Timeline Scrubbing**: When an AI gets stuck in a wall, you can open the VisLog, scrub to the exact millisecond the AI chose its path, and see the red arrow pointing into the wall.
- **Performance**: `UE_VLOG` macros compile out entirely in Shipping builds. They have zero overhead in production.

## Common Mistakes (BAD vs GOOD)

**BAD (Spamming DrawDebugLine)**:
```cpp
DrawDebugLine(GetWorld(), Start, End, FColor::Red, false, 5.0f); // Clutters the screen, disappears after 5 seconds, hard to analyze.
```

**GOOD (Visual Logger)**:
```cpp
UE_VLOG_ARROW(this, LogAI, Log, Start, End, FColor::Red, TEXT("Path")); // Stored in timeline, inspectable forever.
```

## Verification Checklist
- [ ] `#include "VisualLogger/VisualLogger.h"` is present.
- [ ] `UE_VLOG` is used for AI decision logging instead of standard `UE_LOG`.
- [ ] `UE_VLOG_ARROW` or `UE_VLOG_LOCATION` are used to visualize vectors and destinations.
