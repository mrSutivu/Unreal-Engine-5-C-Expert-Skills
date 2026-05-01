---
name: slate-style-sets
description: Defining custom brushes and fonts using FSlateStyleSet. Ensures generated UI elements match the official Unreal Engine aesthetic.
---

# Slate Style Sets (UE5 Expert)

## Activation / When to Use
- Mandatory when building professional Editor plugins or comprehensive Game UIs.
- Use to define global fonts, colors, icons, and button styles.
- Trigger when avoiding hardcoded colors and paths in individual widgets.

## 1. Core Principles
An `FSlateStyleSet` is a dictionary of UI resources (Brushes, Fonts, Colors). Widgets reference the style by name, meaning you can reskin an entire application by simply swapping the active Style Set.

## 2. Implementation Syntax

### Creating the Style Class
Usually implemented as a static singleton within your plugin or UI module.

```cpp
// MyStyle.h
class FMyStyle
{
public:
    static void Initialize();
    static void Shutdown();
    static const ISlateStyle& Get();

private:
    static TSharedPtr<class FSlateStyleSet> StyleInstance;
};
```

### Defining Brushes and Fonts (.cpp)
```cpp
#include "MyStyle.h"
#include "Styling/SlateStyleRegistry.h"
#include "Styling/SlateStyle.h"

TSharedPtr<FSlateStyleSet> FMyStyle::StyleInstance = nullptr;

void FMyStyle::Initialize()
{
    if (!StyleInstance.IsValid())
    {
        StyleInstance = MakeShareable(new FSlateStyleSet("MyProjectStyle"));
        
        // Base path for images
        StyleInstance->SetContentRoot(FPaths::ProjectContentDir() / TEXT("UI/Resources"));

        // Define a core color
        StyleInstance->Set("MyProject.PrimaryColor", FLinearColor(0.8f, 0.1f, 0.1f));

        // Define a custom font
        const FTextBlockStyle NormalText = FTextBlockStyle()
            .SetFont(FSlateFontInfo(FPaths::ProjectContentDir() / TEXT("UI/Fonts/Roboto-Regular.ttf"), 12))
            .SetColorAndOpacity(FSlateColor(FLinearColor::White));
            
        StyleInstance->Set("MyProject.NormalText", NormalText);

        // Define an Image Brush
        StyleInstance->Set("MyProject.Icon", new FSlateImageBrush(StyleInstance->RootToContentDir(TEXT("Icon_64"), TEXT(".png")), FVector2D(64.f, 64.f)));

        FSlateStyleRegistry::RegisterSlateStyle(*StyleInstance);
    }
}

void FMyStyle::Shutdown()
{
    if (StyleInstance.IsValid())
    {
        FSlateStyleRegistry::UnRegisterSlateStyle(*StyleInstance);
        StyleInstance.Reset();
    }
}

const ISlateStyle& FMyStyle::Get()
{
    return *StyleInstance;
}
```

### Using the Style in Slate
```cpp
SNew(STextBlock)
.Text(FText::FromString("Styled Text"))
.TextStyle(&FMyStyle::Get().GetWidgetStyle<FTextBlockStyle>("MyProject.NormalText"))
```

## 3. Reusing Unreal Engine Styles
When building Editor tools, you should mimic the Engine's native look by using `FAppStyle`.

```cpp
SNew(SBorder)
// Uses the native Unreal Editor border style
.BorderImage(FAppStyle::GetBrush("ToolPanel.GroupBorder")) 
[
    SNew(STextBlock)
    // Uses the native Unreal Editor font
    .Font(FAppStyle::GetFontStyle("PropertyWindow.NormalFont")) 
]
```

## 4. Impact on Safety
- **Memory Safety**: Centralizing fonts and brushes prevents the engine from loading the same texture 100 times into memory for 100 different buttons.
- **Trust & Aesthetic**: A plugin that perfectly mimics the `FAppStyle` looks native and professional, reducing user friction.

## Verification Checklist
- [ ] Style is Registered in `StartupModule` (or GameInstance Init) and Unregistered in `ShutdownModule`.
- [ ] Hardcoded paths are avoided in individual widgets.
- [ ] `FAppStyle::GetBrush()` is used for Editor plugins to match the native theme.
