---
name: native-slistview-stileview
description: Implementing high-performance lists in Slate and UMG using SListView and STileView. Essential for handling thousands of data items without lag.
---

# Native SListView / STileView Implementation (UE5 Expert)

## Activation / When to Use
- Mandatory when displaying large datasets (Inventories, Server Browsers, Asset Lists).
- Trigger when a standard `ScrollBox` + `VerticalBox` causes massive lag due to creating hundreds of widgets.

## 1. Core Principles (Virtualization)
`SListView` (and its UMG wrapper `UListView`) do NOT create a widget for every item in your array. They create only enough widgets to fill the visible screen space (e.g., 10 widgets). As you scroll, they recycle those 10 widgets, feeding them new data. This is called **Virtualization**.

## 2. Implementation in C++ (Slate SListView)

### 1. The Data Array
The list requires an array of `TSharedPtr` or `UObject*` to act as the source data.
```cpp
TArray<TSharedPtr<FMyData>> ItemList;
```

### 2. The Row Generator Delegate
You must provide a function that tells the List how to generate a visual row when a new piece of data comes into view.

```cpp
TSharedRef<ITableRow> FMyEditorPanel::OnGenerateRow(TSharedPtr<FMyData> InItem, const TSharedRef<STableViewBase>& OwnerTable)
{
    return SNew(STableRow<TSharedPtr<FMyData>>, OwnerTable)
    [
        SNew(STextBlock).Text(FText::FromString(InItem->Name))
    ];
}
```

### 3. Constructing the List
```cpp
SAssignNew(MyListView, SListView<TSharedPtr<FMyData>>)
.ItemHeight(24)
.ListItemsSource(&ItemList) // Pointer to your data array
.OnGenerateRow(this, &FMyEditorPanel::OnGenerateRow) // Generation delegate
```

### 4. Updating the Data
When your `ItemList` changes, you MUST tell the ListView to refresh.
```cpp
ItemList.Add(MakeShared<FMyData>("New Item"));
MyListView->RequestListRefresh();
```

## 3. Implementation in UMG (UListView)
When using the UMG wrapper, the logic relies on interfaces.
1. The Data Objects must be `UObject` derivatives.
2. The Row Widget must implement the `UUserObjectListEntry` interface.
3. In C++, bind your Data array to the list and call `AddItem()`.

```cpp
UPROPERTY(meta = (BindWidget))
UListView* InventoryList;

void AddData(UItemData* NewItem)
{
    InventoryList->AddItem(NewItem); // Automatically handles virtualization
}
```

## 4. Impact on Safety
- **Memory/CPU Overflow**: A `VerticalBox` with 5,000 text blocks will freeze the engine. A `ListView` with 5,000 items takes 0.1ms to render.
- **Scalability**: Allows the agent to process and display massive lists of project assets safely in Editor Utility Widgets.

## Common Mistakes (BAD vs GOOD)

**BAD (The ScrollBox Trap)**:
```cpp
// Spawns 1000 widgets in memory. Extremely slow.
for (FMyData Data : MassiveArray) {
    MyScrollBox->AddChild(CreateWidget(Data)); 
}
```

**GOOD (Virtualized List)**:
```cpp
// Spawns ~10 widgets and recycles them. Lightning fast.
MyListView->SetListItems(MassiveArray); 
```

## Verification Checklist
- [ ] `SListView`, `STileView`, or `STreeView` used for collections larger than 20 items.
- [ ] Data array uses `TSharedPtr` (Slate) or `UObject*` (UMG).
- [ ] `RequestListRefresh()` is called when the underlying data array size changes.
- [ ] Widgets are never manually added as children to the ListView.
