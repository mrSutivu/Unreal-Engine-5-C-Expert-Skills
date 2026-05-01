---
name: slate-ui-macro-lexicon
description: A complete lexicon of Slate C++ syntax. Covers TAttribute bindings, FSlateBrush integration, and SAssignNew vs SNew patterns.
---

# Slate UI Macro Lexicon (UE5 Reference)

## Activation / When to Use
- Trigger when you need the exact syntax to build Native Slate UIs without UMG.
- Serves as the ultimate "Memory Dump" for C++ declarative UI.

## 1. The Core Macros

### `SNew`
Creates a shared reference (`TSharedRef`) to a new widget. Usually used "inline" within a hierarchy.
```cpp
SNew(STextBlock)
.Text(FText::FromString("Hello"))
```

### `SAssignNew`
Creates a new widget AND assigns a `TSharedPtr` to it. Used when you need to access the widget later (e.g., changing its text dynamically).
```cpp
TSharedPtr<STextBlock> DynamicText;

SAssignNew(DynamicText, STextBlock)
.Text(FText::FromString("Initial Text"))
```

## 2. Parent-Child Relationship Syntax

### Single Child (Content)
Square brackets `[ ]` are used to define the child of a widget.
```cpp
SNew(SBorder)
[
    SNew(STextBlock) // The Border's only child
]
```

### Multiple Children (Slots)
Widgets like `SVerticalBox` and `SHorizontalBox` use `+ Slot()` or `+ SVerticalBox::Slot()`.
```cpp
SNew(SVerticalBox)

+ SVerticalBox::Slot()
.AutoHeight()
.Padding(5.0f)
[
    SNew(STextBlock).Text(FText::FromString("Top"))
]

+ SVerticalBox::Slot()
.FillHeight(1.0f) // Fills remaining space
[
    SNew(STextBlock).Text(FText::FromString("Bottom"))
]
```

## 3. `TAttribute` Bindings (The Magic of Slate)
Almost every argument in Slate (e.g., `.Text(...)`, `.ColorAndOpacity(...)`) accepts a `TAttribute<T>`. This means it can take a static value OR a dynamic function binding.

### Static Value
Assigned once, never updates automatically.
```cpp
.ColorAndOpacity(FLinearColor::Red)
```

### Delegate Binding (Member Function)
Calls the provided function *every frame* Slate draws. Safe and automatic.
```cpp
// In Header: FSlateColor GetDynamicColor() const;
.ColorAndOpacity(this, &SMyWidget::GetDynamicColor)
```

### Lambda Binding
Perfect for quick logic without polluting the header with functions.
```cpp
.ColorAndOpacity_Lambda([this]() -> FSlateColor {
    return bIsError ? FLinearColor::Red : FLinearColor::Green;
})
```

## 4. Event Binding
Events like `.OnClicked`, `.OnHovered` expect delegates that return an `FReply`.

```cpp
SNew(SButton)
.OnClicked_Lambda([this]() -> FReply {
    ExecuteMyAction();
    // Tells the Engine we handled the click, do not pass it to widgets below this one.
    return FReply::Handled(); 
})
```

## 5. Brushes and Fonts (FAppStyle)
To mimic the native Unreal Engine Editor look, use `FAppStyle`.

### Image / Border Brushes
```cpp
SNew(SBorder)
.BorderImage(FAppStyle::GetBrush("ToolPanel.GroupBorder"))

SNew(SImage)
.Image(FAppStyle::GetBrush("Icons.Warning"))
```

### Fonts
```cpp
SNew(STextBlock)
.Font(FAppStyle::GetFontStyle("HeadingExtraSmall")) // Good for tool headers
.Font(FAppStyle::GetFontStyle("PropertyWindow.NormalFont")) // Good for standard text
```

## 6. The `Construct` Template (For Custom Widgets)
When making your own `SCompoundWidget`.

```cpp
void SMyCustomPanel::Construct(const FArguments& InArgs)
{
    // 1. Retrieve arguments
    FText TitleText = InArgs._TitleText;
    FOnClicked ClickDelegate = InArgs._OnSubmit;

    // 2. Build the hierarchy using ChildSlot
    ChildSlot
    [
        SNew(SVerticalBox)
        + SVerticalBox::Slot()
        .AutoHeight()
        [
            SNew(STextBlock).Text(TitleText)
        ]
        + SVerticalBox::Slot()
        .AutoHeight()
        [
            SNew(SButton)
            .OnClicked(ClickDelegate)
            [
                SNew(STextBlock).Text(FText::FromString("Submit"))
            ]
        ]
    ];
}
```
