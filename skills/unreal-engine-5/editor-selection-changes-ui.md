---
name: editor-selection-changes-ui
description: Scripting the Editor UI to update dynamically based on Content Browser or Viewport selection. Reduces oversight demand.
---

# Responding to Editor Selection Changes (UE5 Expert)

## Activation / When to Use
- Mandatory when building contextual Editor tools (e.g., an "Inspector" panel, a Material auto-assigner).
- Trigger when your UI needs to react instantly when the user clicks an Actor or Asset.

## 1. Core Principles
The Unreal Editor broadcasts global events when selections change. Your UI must bind to these global delegates (`USelection::SelectionChangedEvent` or `FEditorDelegates::OnAssetSelectionChanged`) to remain reactive.

## 2. Binding in C++ (Slate or EditorUtilityWidget)

### Viewport Selection (Actors)
Use the `USelection` global events available via the `GEditor` singleton.

```cpp
#include "Selection.h"
#include "Editor.h"

void UMyEditorWidget::NativeConstruct()
{
    Super::NativeConstruct();

    // Bind to the global selection changed event
    USelection::SelectionChangedEvent.AddUObject(this, &UMyEditorWidget::OnActorSelectionChanged);
}

void UMyEditorWidget::NativeDestruct()
{
    // CRITICAL: Unbind to prevent crash when widget closes
    USelection::SelectionChangedEvent.RemoveAll(this);
    Super::NativeDestruct();
}

void UMyEditorWidget::OnActorSelectionChanged(UObject* NewSelection)
{
    USelection* SelectedActors = GEditor->GetSelectedActors();
    if (AActor* FirstSelected = SelectedActors->GetTop<AActor>())
    {
        // Update UI with actor data
        MyTextBlock->SetText(FText::FromString(FirstSelected->GetName()));
    }
}
```

### Content Browser Selection (Assets)
For reacting to clicks in the Content Browser, use `FEditorDelegates` or the `IContentBrowserSingleton`.

```cpp
#include "Editor.h"

void UMyEditorWidget::NativeConstruct()
{
    // Bind to Asset selection changes
    // (Note: FEditorDelegates are heavily used for global editor state tracking)
}
```
*Note: In modern UE5, it is often preferred to use `FContentBrowserModule` for deep integration.*
```cpp
#include "ContentBrowserModule.h"
#include "IContentBrowserSingleton.h"

void FetchSelectedAssets()
{
    FContentBrowserModule& ContentBrowserModule = FModuleManager::LoadModuleChecked<FContentBrowserModule>("ContentBrowser");
    TArray<FAssetData> SelectedAssets;
    ContentBrowserModule.Get().GetSelectedAssets(SelectedAssets);
    
    // Process selected assets
}
```

## 3. Impact on Safety
- **Memory Leaks**: Global Editor delegates (`USelection`, `FEditorDelegates`) live forever. If a widget binds to them and is destroyed without unbinding, clicking an Actor will crash the entire Editor.
- **Contextual Relevance**: Reduces the cognitive load on the user by only displaying tools relevant to the current selection (e.g., hiding skeletal mesh tools when a sound wave is selected).

## Common Mistakes (BAD vs GOOD)

**BAD (Tick Polling)**:
```cpp
// NEVER do this. Kills Editor performance.
void UMyEditorWidget::NativeTick() {
    if (LastSelected != GEditor->GetSelectedActors()->GetTop<AActor>()) {
        UpdateUI();
    }
}
```

**GOOD (Event-Driven)**:
```cpp
// Efficient and safe
USelection::SelectionChangedEvent.AddUObject(this, &UMyEditorWidget::OnSelectionChanged);
```

## Verification Checklist
- [ ] UI relies on Delegate events, NOT Tick polling.
- [ ] Binds to `USelection::SelectionChangedEvent` for Viewport Actors.
- [ ] Delegates are strictly unbound (`RemoveAll` or `Remove`) in `NativeDestruct` or `ShutdownModule`.
- [ ] Safe casting (`GetTop<T>()`) is used when reading the selection data.
