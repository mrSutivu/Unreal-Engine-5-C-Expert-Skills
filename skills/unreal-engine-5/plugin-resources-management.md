---
name: plugin-resources-management
description: Managing custom icons (.png) and content folders within the plugin structure. Ensures generated tools integrate professionally.
---

# Plugin Resources Management (UE5 Expert)

## Activation / When to Use
- Mandatory when creating a new plugin with UI elements, custom Asset types, or specific content.
- Trigger when you need to load an icon for an Editor toolbar button or register a plugin's `/Content` directory.

## 1. Directory Structure
A professional plugin must maintain a strict directory layout:
```text
MyPlugin/
├── MyPlugin.uplugin
├── Resources/           # Icons and raw images (Editor UI)
│   └── Icon128.png      # Must be exactly 128x128 for the plugin browser
├── Content/             # uassets (Blueprints, Materials)
└── Source/              # C++ Code
```

## 2. Enabling Plugin Content
By default, plugins cannot contain content (`.uasset` files). You must explicitly enable it in the `.uplugin` file.

```json
{
    "FileVersion": 3,
    "FriendlyName": "My Plugin",
    "CanContainContent": true, 
    "IsBetaVersion": false
}
```
Once enabled, assets inside `MyPlugin/Content/` are accessed via the path `/MyPlugin/`.

## 3. Registering UI Icons (Slate Style)
To use `.png` files from the `Resources/` folder in your Editor C++ code, you must create an `FSlateStyleSet`.

```cpp
#include "Styling/SlateStyle.h"
#include "Interfaces/IPluginManager.h"
#include "Styling/SlateStyleRegistry.h"

TSharedPtr<FSlateStyleSet> StyleSet;

void FMyPluginModule::StartupModule()
{
    StyleSet = MakeShareable(new FSlateStyleSet("MyPluginStyle"));
    
    // Point to the Resources folder
    FString ContentDir = IPluginManager::Get().FindPlugin("MyPlugin")->GetBaseDir();
    StyleSet->SetContentRoot(ContentDir / TEXT("Resources"));

    // Register a specific icon (e.g., ToolbarButton.png)
    StyleSet->Set("MyPlugin.ToolbarIcon", new FSlateImageBrush(StyleSet->RootToContentDir(TEXT("ToolbarButton"), TEXT(".png")), FVector2D(40.0f, 40.0f)));

    FSlateStyleRegistry::RegisterSlateStyle(*StyleSet.Get());
}

void FMyPluginModule::ShutdownModule()
{
    FSlateStyleRegistry::UnRegisterSlateStyle(*StyleSet.Get());
    StyleSet.Reset();
}
```

## 4. Impact on Safety & UX
- **Professional Integration**: Using native Slate Styles ensures your tools scale correctly with DPI settings and match the Unreal Engine aesthetic.
- **Resource Leaks**: Failing to `UnRegisterSlateStyle` in `ShutdownModule` will cause memory leaks or crashes when reloading the plugin.

## Common Mistakes (BAD vs GOOD)

**BAD (Hardcoded Absolute Paths)**:
```cpp
FString IconPath = "C:/Projects/MyGame/Plugins/MyPlugin/Resources/Icon.png"; // Breaks on any other PC!
```

**GOOD (Relative Plugin Paths)**:
```cpp
FString BaseDir = IPluginManager::Get().FindPlugin("MyPlugin")->GetBaseDir(); // Safe everywhere
```

## Verification Checklist
- [ ] `CanContainContent` is `true` in `.uplugin` if uassets are used.
- [ ] Plugin icon is exactly `128x128` pixels and named `Icon128.png`.
- [ ] `FSlateStyleSet` is properly registered and unregistered.
- [ ] Paths to resources use `IPluginManager::Get().FindPlugin(...)`.
