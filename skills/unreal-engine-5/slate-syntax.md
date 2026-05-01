---
name: slate-syntax
description: Implementing declarative C++ UI using Slate's nested macro syntax. Essential for building deep-level editor extensions.
---

# Slate Syntax (UE5 Expert)

## Activation / When to Use
- Mandatory when building native Editor windows, custom Details Panels, or high-performance game HUDs without UMG overhead.
- Use when UMG is insufficient or impossible (e.g., modifying the Engine's internal UI).

## 1. Core Principles
Slate is Unreal's custom, platform-agnostic, immediate-mode GUI framework. It uses a declarative syntax based on overloaded C++ macros (`SNew`, `SAssignNew`, `+`).

## 2. Implementation Syntax

### The Declarative Macro
- `SNew(WidgetType)`: Creates a new widget and returns a shared reference.
- `SAssignNew(Pointer, WidgetType)`: Creates a new widget AND assigns it to a variable for later use.
- `+ Slot()` / `+ SVerticalBox::Slot()`: Adds a child widget to a parent.

### Example: Building a simple panel
```cpp
TSharedPtr<STextBlock> MyDynamicText;

// Declarative UI construction
TSharedRef<SWidget> MyUI = 
    SNew(SBorder)
    .BorderImage(FAppStyle::GetBrush("ToolPanel.GroupBorder"))
    .Padding(10.0f)
    [
        SNew(SVerticalBox)
        
        + SVerticalBox::Slot()
        .AutoHeight()
        .Padding(5.0f)
        [
            SNew(STextBlock)
            .Text(FText::FromString("Hello Slate!"))
            .Font(FAppStyle::GetFontStyle("HeadingExtraSmall"))
        ]

        + SVerticalBox::Slot()
        .AutoHeight()
        [
            SAssignNew(MyDynamicText, STextBlock)
            .Text(FText::FromString("Click the button..."))
        ]

        + SVerticalBox::Slot()
        .AutoHeight()
        .Padding(0, 10.0f, 0, 0)
        [
            SNew(SButton)
            .Text(FText::FromString("Submit"))
            .OnClicked_Lambda([this]() {
                MyDynamicText->SetText(FText::FromString("Submitted!"));
                return FReply::Handled();
            })
        ]
    ];
```

## 3. Arguments and Attributes (TAttribute)
Slate heavily uses `TAttribute<T>`, which allows a property to either be a static value OR a dynamic function binding.

```cpp
// Static Value
.Text(FText::FromString("Static Text"))

// Dynamic Binding (Called every frame Slate draws)
.Text_Lambda([]() { return FText::FromString(FDateTime::Now().ToString()); })

// Delegate Binding
.Text(this, &SMyWidget::GetDynamicText)
```

## 4. Impact on Safety
- **Memory Safety**: Slate uses Unreal's `TSharedPtr` / `TSharedRef` exclusively. Widgets are destroyed automatically when no longer referenced.
- **Performance**: Bypasses the `UObject` overhead entirely. Millions of Slate widgets can be rendered efficiently.

## Common Mistakes (BAD vs GOOD)

**BAD (Missing brackets)**:
```cpp
SNew(SVerticalBox)
+ SVerticalBox::Slot()
SNew(STextBlock) // ERROR: Syntax error, missing brackets around child
```

**GOOD (Proper nesting)**:
```cpp
SNew(SVerticalBox)
+ SVerticalBox::Slot()
[
    SNew(STextBlock)
]
```

## Verification Checklist
- [ ] `SNew` or `SAssignNew` used to instantiate widgets.
- [ ] Square brackets `[ ]` used to define children.
- [ ] `TSharedPtr` and `TSharedRef` used for storage (Never raw pointers for Slate).
- [ ] `FReply::Handled()` returned from interaction events like `OnClicked`.
