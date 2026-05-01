---
name: scompoundwidget
description: Building reusable, custom Slate components via SCompoundWidget to maintain a consistent visual language across custom plugins.
---

# SCompoundWidget (UE5 Expert)

## Activation / When to Use
- Mandatory when you need to encapsulate a complex Slate UI into a reusable, self-contained component.
- Use to create standardized custom inputs, custom lists, or complex toolbars.

## 1. Core Principles
`SCompoundWidget` is the base class for creating custom Slate widgets. It allows you to define custom arguments (`SLATE_ARGUMENT`), events (`SLATE_EVENT`), and a `Construct` function.

## 2. Implementation Syntax

### Header (.h)
```cpp
#pragma once
#include "CoreMinimal.h"
#include "Widgets/SCompoundWidget.h"

// Define the custom widget
class SMyCustomButton : public SCompoundWidget
{
public:
    // The macro block that defines initialization arguments
    SLATE_BEGIN_ARGS(SMyCustomButton)
        : _ButtonText(FText::FromString("Default Text")) // Default value
    {}
        // Define arguments (passed during SNew)
        SLATE_ARGUMENT(FText, ButtonText)
        SLATE_EVENT(FOnClicked, OnClicked)
    SLATE_END_ARGS()

    // Required Construct function
    void Construct(const FArguments& InArgs);

private:
    FOnClicked OnClickedDelegate;
};
```

### Source (.cpp)
```cpp
#include "SMyCustomButton.h"
#include "Widgets/Input/SButton.h"
#include "Widgets/Text/STextBlock.h"

void SMyCustomButton::Construct(const FArguments& InArgs)
{
    // Store delegates or arguments locally if needed
    OnClickedDelegate = InArgs._OnClicked;

    // Define the visual hierarchy using ChildSlot
    ChildSlot
    [
        SNew(SButton)
        .OnClicked(OnClickedDelegate)
        .ContentPadding(FMargin(10.0f, 5.0f))
        [
            SNew(STextBlock)
            .Text(InArgs._ButtonText)
            .Font(FAppStyle::GetFontStyle("PropertyWindow.NormalFont"))
        ]
    ];
}
```

## 3. Usage inside other Slate code
Once defined, you instantiate your custom widget using `SNew` just like any native Slate widget.

```cpp
SNew(SVerticalBox)
+ SVerticalBox::Slot()
[
    SNew(SMyCustomButton)
    .ButtonText(FText::FromString("Launch Tool"))
    .OnClicked(this, &FMyEditorModule::OnToolLaunched)
]
```

## 4. Impact on Safety
- **Encapsulation**: Keeps main Editor Window code clean by abstracting complex rows or buttons into their own isolated classes.
- **Consistency**: Ensures the agent can generate a UI toolkit that looks identical across multiple plugins by reusing the same `SCompoundWidget` foundations.

## Common Mistakes (BAD vs GOOD)

**BAD (Forgetting ChildSlot)**:
```cpp
void SMyCustomButton::Construct(const FArguments& InArgs) {
    // Missing ChildSlot! The widget will be invisible.
    SNew(SButton); 
}
```

**GOOD (Using ChildSlot)**:
```cpp
void SMyCustomButton::Construct(const FArguments& InArgs) {
    ChildSlot [ SNew(SButton) ]; // Correct
}
```

## Verification Checklist
- [ ] Class inherits from `SCompoundWidget`.
- [ ] `SLATE_BEGIN_ARGS` and `SLATE_END_ARGS` block defines all inputs.
- [ ] Arguments in the CPP file are accessed via `InArgs._ArgumentName`.
- [ ] Visual hierarchy is built inside the `ChildSlot [ ]` array.
