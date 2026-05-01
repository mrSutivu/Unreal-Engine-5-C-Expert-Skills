---
name: customizing-outliner-content-browser
description: Using C++ to filter editor views and modify selection behaviors in the Outliner and Content Browser.
---

# Customizing the Outliner & Content Browser (UE5 Expert)

## Activation / When to Use
- Trigger when building complex Editor plugins that manage large scenes.
- Use to help humans/agents quickly locate specific "Artifacts" or missing dependencies.

## 1. Outliner Customization (Scene Outliner)
You can extend the Scene Outliner by adding custom columns. This is done via `ISceneOutlinerModule`.

### Adding a Custom Column
This requires an Editor module.
```cpp
#include "ISceneOutliner.h"
#include "SceneOutlinerModule.h"

void FMyEditorModule::StartupModule()
{
    FSceneOutlinerModule& SceneOutlinerModule = FModuleManager::LoadModuleChecked<FSceneOutlinerModule>("SceneOutliner");
    
    FSceneOutlinerColumnInfo ColumnInfo(
        ISceneOutlinerColumn::FCreateSceneOutlinerColumn::CreateRaw(this, &FMyEditorModule::CreateCustomColumn),
        FDelegateGetColumnVisibility::CreateLambda([]() { return true; }),
        FSceneOutlinerColumnInfo::EVisibility::Visible
    );

    SceneOutlinerModule.RegisterDefaultColumnType<FMyCustomColumn>(ColumnInfo);
}
```

## 2. Content Browser Filtering
You can create custom Asset Filters (Frontend Filters) to help users find specific setups (e.g., "Materials missing physical surfaces").

### Creating a Frontend Filter
```cpp
#include "FrontendFilterBase.h"

class FFrontendFilter_MissingPhysics : public FFrontendFilter
{
public:
    virtual FString GetName() const override { return TEXT("Missing Physics"); }
    virtual FText GetDisplayName() const override { return LOCTEXT("FilterMissingPhysics", "Missing Physics"); }
    virtual FText GetToolTipText() const override { return LOCTEXT("FilterMissingPhysicsTooltip", "Show materials without physics assignment"); }
    
    // Core logic
    virtual bool PassesFilter(FAssetFilterType InItem) const override
    {
        if (UObject* Asset = InItem.GetAsset())
        {
            if (UMaterial* Mat = Cast<UMaterial>(Asset))
            {
                return Mat->GetPhysicalMaterial() == nullptr;
            }
        }
        return false;
    }
};
```

## 3. Managing Selections
Use `GEditor` to interact with what the user has currently selected in the viewport or outliner.

```cpp
#include "Editor.h"
#include "Engine/Selection.h"

void ProcessSelection()
{
    USelection* SelectedActors = GEditor->GetSelectedActors();
    for (FSelectionIterator It(*SelectedActors); It; ++It)
    {
        if (AActor* Actor = Cast<AActor>(*It))
        {
            // Process actor
        }
    }
}
```

## 4. Impact on Safety
- **Cognitive Load**: Custom filters reduce the time spent searching through thousands of assets, lowering "Oversight Demand."
- **Data Validation**: Custom outliner columns can display validation states (Red/Green indicators) instantly, catching errors before runtime.

## Verification Checklist
- [ ] Outliner column registration is done in `StartupModule` and unregistered in `ShutdownModule`.
- [ ] Frontend filters execute quickly (`PassesFilter` must be optimized).
- [ ] `GEditor->GetSelectedActors()` is used safely (checking for nulls and casting properly).
