---
name: deep-editor-customization
description: Comprehensive templates for creating Custom Asset Types, Factories, and Custom Details Panels in UE5.
---

# Deep Editor Customization Templates (UE5 Expert Reference)

## Activation / When to Use
- Trigger when generating a new custom file format in the Content Browser or overhauling an Actor's Details Panel.
- Serves as the "Memory Dump" for Editor Extensibility.

## 1. Custom Asset Type Template
Requires separating code into a `Runtime` module (for the Data) and an `Editor` module (for the UI/Factory).

### A. The Data Object (`Runtime` Module)
```cpp
// MyCustomAsset.h
#pragma once
#include "CoreMinimal.h"
#include "UObject/NoExportTypes.h"
#include "MyCustomAsset.generated.h"

UCLASS(BlueprintType)
class MYRUNTIME_API UMyCustomAsset : public UObject
{
    GENERATED_BODY()
public:
    UPROPERTY(EditAnywhere, Category = "Data")
    FString ConfigurationData;
};
```

### B. The Factory (`Editor` Module)
Allows right-click creation in the Content Browser.
```cpp
// MyCustomAssetFactory.h
#pragma once
#include "Factories/Factory.h"
#include "MyCustomAssetFactory.generated.h"

UCLASS()
class MYEDITOR_API UMyCustomAssetFactory : public UFactory
{
    GENERATED_BODY()
public:
    UMyCustomAssetFactory()
    {
        SupportedClass = UMyCustomAsset::StaticClass();
        bCreateNew = true;
        bEditAfterNew = true;
    }

    virtual UObject* FactoryCreateNew(UClass* Class, UObject* InParent, FName Name, EObjectFlags Flags, UObject* Context, FFeedbackContext* Warn) override
    {
        return NewObject<UMyCustomAsset>(InParent, Class, Name, Flags);
    }
};
```

### C. The Asset Actions (`Editor` Module)
Defines UI color, name, and categories.
```cpp
// MyCustomAssetActions.h
#pragma once
#include "AssetTypeActions_Base.h"
#include "MyCustomAsset.h"

class FMyCustomAssetActions : public FAssetTypeActions_Base
{
public:
    virtual FText GetName() const override { return FText::FromString("My Custom Asset"); }
    virtual FColor GetTypeColor() const override { return FColor::Emerald; }
    virtual UClass* GetSupportedClass() const override { return UMyCustomAsset::StaticClass(); }
    virtual uint32 GetCategories() override { return EAssetTypeCategories::Misc; }
};
```

### D. Registration (`Editor` Module Startup)
```cpp
// MyEditorModule.cpp
#include "MyCustomAssetActions.h"
#include "IAssetTools.h"
#include "AssetToolsModule.h"

TSharedPtr<FMyCustomAssetActions> CustomAssetActions;

void FMyEditorModule::StartupModule()
{
    IAssetTools& AssetTools = FModuleManager::LoadModuleChecked<FAssetToolsModule>("AssetTools").Get();
    CustomAssetActions = MakeShareable(new FMyCustomAssetActions());
    AssetTools.RegisterAssetTypeActions(CustomAssetActions.ToSharedRef());
}

void FMyEditorModule::ShutdownModule()
{
    if (FModuleManager::Get().IsModuleLoaded("AssetTools"))
    {
        IAssetTools& AssetTools = FModuleManager::GetModuleChecked<FAssetToolsModule>("AssetTools").Get();
        AssetTools.UnregisterAssetTypeActions(CustomAssetActions.ToSharedRef());
    }
}
```

## 2. Custom Details Panel Template
Completely overrides how properties are displayed.

### A. The Customization Class (`Editor` Module)
```cpp
// MyDetailsCustomization.h
#pragma once
#include "IDetailCustomization.h"

class FMyDetailsCustomization : public IDetailCustomization
{
public:
    static TSharedRef<IDetailCustomization> MakeInstance();
    virtual void CustomizeDetails(IDetailLayoutBuilder& DetailBuilder) override;
};
```

### B. The Implementation
```cpp
// MyDetailsCustomization.cpp
#include "MyDetailsCustomization.h"
#include "DetailLayoutBuilder.h"
#include "DetailCategoryBuilder.h"
#include "DetailWidgetRow.h"
#include "Widgets/Input/SButton.h"
#include "MyRuntimeActor.h"

TSharedRef<IDetailCustomization> FMyDetailsCustomization::MakeInstance()
{
    return MakeShareable(new FMyDetailsCustomization);
}

void FMyDetailsCustomization::CustomizeDetails(IDetailLayoutBuilder& DetailBuilder)
{
    IDetailCategoryBuilder& Category = DetailBuilder.EditCategory("Custom Tools");

    Category.AddCustomRow(FText::FromString("Magic Button"))
    .NameContent()
    [
        SNew(STextBlock).Text(FText::FromString("Actions"))
    ]
    .ValueContent()
    [
        SNew(SButton)
        .Text(FText::FromString("Click Me!"))
        .OnClicked_Lambda([&DetailBuilder]() -> FReply
        {
            TArray<TWeakObjectPtr<UObject>> Objects;
            DetailBuilder.GetObjectsBeingCustomized(Objects);
            for (auto Obj : Objects)
            {
                if (AMyRuntimeActor* Actor = Cast<AMyRuntimeActor>(Obj.Get()))
                {
                    Actor->ExecuteMagic();
                }
            }
            return FReply::Handled();
        })
    ];
}
```

### C. Registration (`Editor` Module Startup)
```cpp
// MyEditorModule.cpp
#include "PropertyEditorModule.h"

void FMyEditorModule::StartupModule()
{
    FPropertyEditorModule& PropertyModule = FModuleManager::LoadModuleChecked<FPropertyEditorModule>("PropertyEditor");
    PropertyModule.RegisterCustomClassLayout(
        "MyRuntimeActor", 
        FOnGetDetailCustomizationInstance::CreateStatic(&FMyDetailsCustomization::MakeInstance)
    );
}
```
