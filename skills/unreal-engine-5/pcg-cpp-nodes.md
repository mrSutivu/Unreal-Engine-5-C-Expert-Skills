---
name: pcg-cpp-nodes
description: Creating custom Procedural Content Generation (PCG) C++ nodes (UPCGBlueprintElement / UPCGCustomAction).
---

# PCG Custom C++ Nodes (UE5 Expert)

## Activation / When to Use
- Mandatory when the visual Procedural Content Generation (PCG) graph cannot perform a complex math algorithm efficiently.
- Trigger when you need to read custom C++ data structures (e.g., custom voxel grids, external AI maps) and feed them into PCG points.

## 1. Core Principles
PCG processes points in space. A custom node takes an input collection of points (`FPCGPoint`), modifies them, and outputs a new collection.

## 2. Implementation Syntax
In modern UE5 PCG, you create custom nodes by subclassing `UPCGSettings` and `FPCGContext`.

### 1. The Settings Class (The Node UI)
Defines the input/output pins and properties visible in the Graph.

```cpp
#pragma once
#include "PCGSettings.h"
#include "MyPCGNode.generated.h"

UCLASS(BlueprintType, ClassGroup = (Procedural))
class UMyPCGSettings : public UPCGSettings
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, Category = "Settings")
    float MultiplyFactor = 2.0f;

    // Defines the execution method
    virtual FPCGElementPtr CreateElement() const override;

    // Defines Input/Output Pins
    virtual TArray<FPCGPinProperties> InputPinProperties() const override;
    virtual TArray<FPCGPinProperties> OutputPinProperties() const override;
};
```

### 2. The Element Class (The Logic)
Executes the math asynchronously.

```cpp
#include "PCGContext.h"
#include "Elements/PCGExecuteAction.h" // For FPCGElement

class FMyPCGElement : public FSimplePCGElement // Simplified element base
{
protected:
    virtual bool ExecuteInternal(FPCGContext* Context) const override;
};
```

### 3. The Execution Logic
```cpp
FPCGElementPtr UMyPCGSettings::CreateElement() const { return MakeShared<FMyPCGElement>(); }

bool FMyPCGElement::ExecuteInternal(FPCGContext* Context) const
{
    const UMyPCGSettings* Settings = Context->GetInputSettings<UMyPCGSettings>();
    check(Settings);

    // 1. Get Input Data
    TArray<FPCGTaggedData> Inputs = Context->InputData.GetInputs();
    TArray<FPCGTaggedData>& Outputs = Context->OutputData.TaggedData;

    for (const FPCGTaggedData& Input : Inputs)
    {
        const UPCGSpatialData* SpatialData = Cast<UPCGSpatialData>(Input.Data);
        if (!SpatialData) continue;

        // Extract points
        const UPCGPointData* PointData = SpatialData->ToPointData(Context);
        if (!PointData) continue;

        // 2. Create Output Container
        UPCGPointData* OutPointData = NewObject<UPCGPointData>();
        OutPointData->InitializeFromData(PointData);
        TArray<FPCGPoint>& OutPoints = OutPointData->GetMutablePoints();

        // 3. Process Points
        for (FPCGPoint& Point : OutPoints)
        {
            Point.Transform.SetScale3D(Point.Transform.GetScale3D() * Settings->MultiplyFactor);
        }

        // 4. Output Data
        FPCGTaggedData& Output = Outputs.Add_GetRef(Input);
        Output.Data = OutPointData;
    }

    return true; // Execution succeeded
}
```

## 3. Impact on Safety
- **Async Threading**: `ExecuteInternal` is called on a worker thread by the PCG subsystem. You MUST NOT spawn Actors or call functions that require the Game Thread inside this loop.
- **Memory Allocation**: Using `OutPointData->InitializeFromData(PointData)` ensures the new points inherit the metadata (color, density) of the original points safely without manual copying.

## Verification Checklist
- [ ] `PCG` module is added to `Build.cs`.
- [ ] Class is split into `UPCGSettings` (Data) and `FPCGElement` (Logic).
- [ ] `ExecuteInternal` does not contain Game-Thread-only calls.
- [ ] Output `FPCGTaggedData` correctly references the newly allocated `UPCGPointData`.
