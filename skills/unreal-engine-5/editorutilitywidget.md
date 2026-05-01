---
name: editorutilitywidget
description: Designing dockable UI panels for the Unreal Editor using EditorUtilityWidgets. Enables visual feedback and tool interfaces.
---

# EditorUtilityWidget (UE5 Expert)

## Activation / When to Use
- Mandatory when creating custom Editor windows, toolbars, or dockable panels.
- Use to provide users/agents with visual feedback (progress bars, logs, buttons) without writing pure Slate C++.

## 1. Core Principles
An `EditorUtilityWidget` (EUW) is a UMG widget that runs exclusively in the Editor. It has access to Editor-only Subsystems and can be docked like any native Unreal panel.

## 2. Implementation (C++ Backed)
For complex tools, always back your EUW with a C++ class to handle heavy logic, keeping the Blueprint graph clean.

### C++ Base Class (.h)
Inherit from `UEditorUtilityWidget`.
```cpp
#pragma once
#include "CoreMinimal.h"
#include "EditorUtilityWidget.h"
#include "MyCustomEditorTool.generated.h"

UCLASS()
class MYEDITOR_API UMyCustomEditorTool : public UEditorUtilityWidget
{
    GENERATED_BODY()

public:
    UFUNCTION(BlueprintCallable, Category = "Tool Logic")
    void ExecuteHeavyTask();

protected:
    // Bound to a button in the UMG designer
    UPROPERTY(meta = (BindWidget))
    class UButton* StartTaskButton;

    virtual void NativeConstruct() override;
};
```

## 3. Spawning and Docking
EUWs can be spawned via right-click in the Content Browser ("Run Editor Utility Widget"), or programmatically via C++ using the `UEditorUtilitySubsystem`.

```cpp
#include "EditorUtilitySubsystem.h"
#include "EditorUtilityWidgetBlueprint.h"

void FMyEditorModule::SpawnToolWindow()
{
    UEditorUtilitySubsystem* EditorUtilitySubsystem = GEditor->GetEditorSubsystem<UEditorUtilitySubsystem>();
    
    // Load the blueprint asset
    UEditorUtilityWidgetBlueprint* WidgetBP = LoadObject<UEditorUtilityWidgetBlueprint>(nullptr, TEXT("/Game/EditorTools/EUW_MyTool.EUW_MyTool"));
    
    if (WidgetBP && EditorUtilitySubsystem)
    {
        EditorUtilitySubsystem->SpawnAndRegisterTab(WidgetBP);
    }
}
```

## 4. Impact on Safety
- **Non-Destructive**: Runs safely in the editor environment. Cannot crash the packaged game.
- **Oversight**: Provides a visual "Manager Surface" where agents can report status (e.g., "Compiling Shaders: 45%") clearly to the user.

## Common Mistakes (BAD vs GOOD)

**BAD (Runtime Widget for Editor)**:
```text
Creating a standard `UUserWidget` and trying to spawn it via `CreateWidget` in an Editor plugin. It will not dock and lacks Editor access.
```

**GOOD (Editor Utility Widget)**:
```text
Inheriting from `UEditorUtilityWidget` and using `SpawnAndRegisterTab` to integrate seamlessly into the Editor layout.
```

## Verification Checklist
- [ ] Parent class is `UEditorUtilityWidget`.
- [ ] Asset type in Content Browser is `Editor Utility Widget`.
- [ ] Spawning is handled via `UEditorUtilitySubsystem`.
- [ ] Heavy logic is offloaded to C++ using `BindWidget`.
