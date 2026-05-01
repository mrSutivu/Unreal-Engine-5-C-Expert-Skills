---
name: custom-blueprint-nodes-uk2node
description: Creating complex, dynamic Blueprint nodes by subclassing UK2Node. Allows changing pins dynamically and wildcards.
---

# Custom Blueprint Nodes (UK2Node) (UE5 Expert)

## Activation / When to Use
- Mandatory when a simple `UFUNCTION(BlueprintCallable)` cannot handle the logic (e.g., you need "Wildcard" pins that change type depending on what is connected, like the `SpawnActorFromClass` node).
- Trigger when you need to hide/show execution pins dynamically.

## 1. Core Principles
`UK2Node` is the base class for all Blueprint visual nodes. These nodes exist ONLY in the Editor. When the Blueprint is compiled, the `UK2Node` "expands" into standard C++ function calls (Thunks).

## 2. The Implementation Flow
Creating a `UK2Node` requires heavy boilerplate.

### 1. The Node Class (`Editor Module`)
```cpp
#pragma once
#include "K2Node.h"
#include "MyK2Node_DynamicSpawner.generated.h"

UCLASS()
class MYEDITOR_API UMyK2Node_DynamicSpawner : public UK2Node
{
    GENERATED_BODY()

public:
    // 1. UI Information
    virtual FText GetNodeTitle(ENodeTitleType::Type TitleType) const override;
    virtual FText GetTooltipText() const override;
    virtual FSlateIcon GetIconAndTint(FLinearColor& OutColor) const override;

    // 2. Pin Creation
    virtual void AllocateDefaultPins() override;

    // 3. Dynamic Pin Logic (When a connection changes)
    virtual void PinConnectionListChanged(UEdGraphPin* Pin) override;

    // 4. Compilation (Translating node to C++ calls)
    virtual void ExpandNode(class FKismetCompilerContext& CompilerContext, UEdGraph* SourceGraph) override;
};
```

### 2. The Thunk Function (`Runtime Module`)
The `UK2Node` needs a real C++ function to point to during compilation.

```cpp
// In a UBlueprintFunctionLibrary
UFUNCTION(BlueprintCallable, meta=(BlueprintInternalUseOnly = "true"))
static UObject* Internal_SpawnDynamicObject(UClass* ObjectClass);
```

### 3. Expanding the Node
During compilation, you wire the visual pins to the Thunk function.

```cpp
void UMyK2Node_DynamicSpawner::ExpandNode(FKismetCompilerContext& CompilerContext, UEdGraph* SourceGraph)
{
    Super::ExpandNode(CompilerContext, SourceGraph);

    // 1. Find the internal C++ function
    UFunction* InternalFunc = UMyBlueprintLibrary::StaticClass()->FindFunctionByName(GET_FUNCTION_NAME_CHECKED(UMyBlueprintLibrary, Internal_SpawnDynamicObject));

    // 2. Create an intermediate node to call the function
    UK2Node_CallFunction* CallFuncNode = CompilerContext.SpawnIntermediateNode<UK2Node_CallFunction>(this, SourceGraph);
    CallFuncNode->SetFromFunction(InternalFunc);
    CallFuncNode->AllocateDefaultPins();

    // 3. Move the connections from our custom visual node to the hidden function node
    CompilerContext.MovePinLinksToIntermediate(*GetExecPin(), *CallFuncNode->GetExecPin());
    CompilerContext.MovePinLinksToIntermediate(*GetClassPin(), *CallFuncNode->FindPinChecked(TEXT("ObjectClass")));
    
    // 4. Destroy the visual node (it's no longer needed for runtime)
    BreakAllNodeLinks();
}
```

## 3. Impact on Safety
- **Editor-Only Constraint**: `UK2Node`s MUST be in an Editor module. If they leak into Runtime, the game will fail to package.
- **Thunk Security**: The Thunk function must be marked `BlueprintInternalUseOnly = "true"` so designers don't see it directly in the right-click menu.

## Verification Checklist
- [ ] Node inherits from `UK2Node`.
- [ ] Lives exclusively in an Editor module.
- [ ] `AllocateDefaultPins` is used to spawn Exec and Data pins.
- [ ] `ExpandNode` correctly maps all visual pins to a `BlueprintInternalUseOnly` C++ function.
