# 网格背包使用与扩展文档

本文面向需要在 Unity 项目中接入、维护、扩展 Cholopol Tetris Inventory System 的开发者。它按实际代码结构整理，重点说明“怎么用”和“改哪里”。

## 1. 系统定位

这个系统是一个类似《逃离塔科夫》的网格背包：

- 物品占用一个或多个格子，形状由坐标点集定义。
- 物品可以旋转，旋转后重新计算占用格子和显示尺寸。
- 物品可以放入普通网格、装备槽位，也可以拥有自己的内部网格。
- 拖拽时使用 Ghost 物体预览目标位置，并用高亮颜色反馈可放置、不可放置、可堆叠、可快速交换。
- 运行时状态会写入 `InventoryData_SO`，存档系统再把这些数据保存到存档槽。

核心代码位于：

- `Assets/Cholopol_Tetris_Inventory_System/Runtime`
- `Assets/Cholopol_Tetris_Inventory_System/Config`
- `Assets/Cholopol_Tetris_Inventory_System/Editor`

示例场景位于：

- `Assets/Cholopol_Tetris_Inventory_System_Samples/Demo/Scenes/EFT Like UI.unity`

## 2. 基本运行依赖

项目依赖以下 Unity 包和框架：

- Unity 2022.3 LTS。
- Loxodon Framework：用于 MVVM 绑定和 Window 系统。
- Unity Localization：用于物品名称、描述本地化。
- Newtonsoft Json：用于存档序列化。
- Addressables：用于预制体配置和异步加载。

场景里必须存在或正确配置这些对象：

- `LoxodonInitializer`：注册 Loxodon Binding、`IInventoryService`、`IInventoryTreeCache` 等服务。
- `InventoryManager`：全局输入、当前鼠标下网格、Ghost 旋转、高亮显示。
- `InventorySaveLoadService`：持久化数据、预留网格/槽位恢复、运行时缓存构建。
- `PoolManager`：高亮块、物品 UI 等对象复用。
- `SaveLoadManager`：存档槽读写。
- `EventSystem` 和 Canvas。

## 3. 核心概念

### 3.1 ItemDetails：物品静态配置

定义位置：`Runtime/Core/Data/DataCollections.cs`

`ItemDetails` 是一个物品类型的静态配置，保存在 `ItemDataList_SO` 中。常用字段：

- `itemID`：物品类型 ID，运行时和存档通过它找回静态配置。
- `localizedName` / `localizedDescription`：本地化名称和描述。
- `tetrisPieceShape`：物品形状枚举。
- `itemIcon`：物品 UI 图标。
- `gridUIPrefab`：物品自带内部网格时使用的网格面板预制体。
- `inventorySlotType`：可放入的装备槽类型。
- `itemRarity`：稀有度，用于背景色。
- `maxStack`：最大堆叠数量。大于 0 时参与堆叠逻辑。
- `xWidth` / `yHeight`：物品未旋转时的外接矩形宽高。
- `dir`：初始方向。

扩展物品属性时，优先给 `ItemDetails` 添加静态字段；运行中会变化的字段放到 `TetrisItemPersistentData.CustomData` 或另建存档结构。

### 3.2 PointSet：物品形状

定义位置：`Runtime/Core/Data/DataCollections.cs`

`PointSet` 用一组 `Vector2Int` 表示物品占用格子。坐标以物品锚点为原点，例如 2x2 正方形可以是：

```csharp
new List<Vector2Int>
{
    new Vector2Int(0, 0),
    new Vector2Int(1, 0),
    new Vector2Int(0, 1),
    new Vector2Int(1, 1),
}
```

所有形状保存在 `TetrisItemPointSet_SO` 的 `TetrisPieceShapeList` 中，并通过 `TetrisPieceShape` 枚举下标访问：

```csharp
InventoryManager.Instance.GetTetrisCoordinateSet(shape);
```

注意：`TetrisPieceShape` 枚举顺序必须和 `TetrisItemPointSet_SO.TetrisPieceShapeList` 顺序保持一致。

### 3.3 TetrisItemPersistentData：物品运行时/存档数据

定义位置：`Runtime/Core/Data/DataCollections.cs`

这部分记录单个物品实例的动态状态：

- `itemID`：对应 `ItemDetails.itemID`。
- `itemGuid`：物品实例唯一 ID。
- `orginPosition`：物品在网格内的左上锚点坐标。
- `direction`：当前方向。
- `stack`：当前堆叠数量。
- `parentItemGuid`：如果物品在另一个物品内部网格中，这里记录父物品 GUID。
- `persistentGridGuid`：所在持久化网格 GUID。
- `isOnSlot` / `slotIndex`：是否在装备槽，以及槽位索引。
- `gridPIndex`：物品内部第几个网格。
- `CustomData`：预留自定义数据。

`TetrisItemVM.SetItemData()` 会把 VM 当前状态写回 `InventoryData_SO`。

### 3.4 TetrisGridVM：网格逻辑

定义位置：`Runtime/MVVM/VM/TetrisGridVM.cs`

`TetrisGridVM` 是网格背包的核心逻辑类，负责：

- 网格尺寸和单格像素尺寸：`GridSizeWidth`、`GridSizeHeight`、`LocalGridTileSizeWidth`、`LocalGridTileSizeHeight`。
- 占用矩阵：`TetrisItemOccupiedCells[x, y]`。
- 当前网格拥有的物品：`OwnerItemsDic`。
- 判断边界、重叠、空位、快速交换。
- 触发 View 生成或移除物品 UI。

常用方法：

```csharp
grid.TryPlaceTetrisItem(itemVm, x, y);
grid.PlaceTetrisItem(itemVm, x, y);
grid.RemoveTetrisItem(itemVm, x, y, itemVm.RotationOffset, itemVm.TetrisCoordinateSet);
grid.IsAreaVacantForItem(itemVm, x, y);
grid.BoundryCheck(x, y, itemVm.Width, itemVm.Height);
grid.GetTetrisItemVM(x, y);
```

通常优先调用 `TryPlaceTetrisItem`，除非你已经在外部做完全部校验。

### 3.5 TetrisItemVM：物品逻辑

定义位置：`Runtime/MVVM/VM/TetrisItemVM.cs`

`TetrisItemVM` 是一个物品实例的 ViewModel，负责：

- 物品静态配置：`ItemDetails`。
- 当前容器：`CurrentTetrisContainer`。
- 当前方向和旋转状态：`Direction`、`Rotated`。
- 当前形状点集：`TetrisCoordinateSet`。
- 旋转补偿：`RotationOffset`。
- 当前尺寸：`Width`、`Height`、`Size`。
- 堆叠数量：`CurrentStack`。
- 自带内部网格：`OwnedTetrisGridsVM`。

常用方法：

```csharp
itemVm.Rotate();
itemVm.UpdateSize(containerVm);
itemVm.SetItemData();
itemVm.RemoveItemData();
itemVm.GetOrCreateGridVM(index);
```

旋转逻辑由 `TetrisUtilities.RotationHelper` 处理。改变 `Direction` 后会重新计算点集、旋转补偿和显示尺寸。

### 3.6 TetrisSlotVM：装备槽逻辑

定义位置：`Runtime/MVVM/VM/TetrisSlotVM.cs`

`TetrisSlotVM` 用于单格装备槽，不按网格点集判断占用，而是判断：

- 槽内是否已有物品。
- 物品 `SlotType` 是否和槽位类型匹配。

实际放置规则还会经过 `InventoryPlacementConfig_SO` 或 `IInventoryService.CanPlace`。

### 3.7 View：UI 绑定和事件转发

主要 View：

- `TetrisGridView`：绑定 `TetrisGridVM`，根据 VM 生成和摆放物品 UI。
- `TetrisItemView`：显示图标、稀有度背景、名称、堆叠数，并初始化物品内部网格。
- `TetrisItemGhostView`：拖拽预览物体，接收拖拽和旋转交互。
- `TetrisSlotView`：装备槽 UI。

View 层不应承载复杂规则。扩展规则优先改 Service、VM 或配置。

## 4. 快速接入一个新场景

### 4.1 放入基础管理对象

在场景中准备：

1. Canvas。
2. EventSystem。
3. `LoxodonInitializer`。
4. `PoolManager`。
5. `InventoryManager`。
6. `InventorySaveLoadService`。
7. `SaveLoadManager`。
8. `TetrisItemGhostView` 对象。
9. `InventoryHighlight` 对象。

可以直接参考示例场景复制一套，再替换数据资产和 UI。

### 4.2 配置 InventoryManager

`InventoryManager` Inspector 中关键引用：

- `itemDataList_SO`：物品数据库。
- `tetrisItemPointSet_SO`：形状数据库。
- `depositoryGrid` / `depositoryGridView`：主仓库网格。
- `canvas`：背包 UI 所在 Canvas。
- `inventoryHighlight`：高亮组件。
- `tetrisItemGhost`：拖拽 Ghost。
- `rightClickMenuPanel`：右键菜单。
- `placementConfig`：放置规则配置，可为空，空时使用默认规则。
- `inventorySystemRoot`：背包 UI 根对象，用于 `B` 键打开/关闭。
- `startPanel`：背包关闭时显示的起始面板，可选。

默认输入：

- `B`：打开/关闭背包 UI。
- `R`：拖拽物品时旋转 Ghost。
- 鼠标左键：拖拽。
- 鼠标右键：右键菜单。

### 4.3 配置持久化网格

每个需要保存物品的网格对象应具备：

- `TetrisGridView`
- `DataGUID`
- 可选 `InventoryGridDescriptor`

`DataGUID.guid` 是持久化关键，不要在已有存档项目中随意修改。

如果这个网格属于死亡后保留区域，例如保险箱：

```csharp
InventoryGridDescriptor.category = PersistentGridType.Coffer;
InventoryGridDescriptor.retainedOnDeath = true;
```

`InventorySaveLoadService.inventoryPersistentGrids` 中应填入这些预留网格。

### 4.4 配置装备槽

装备槽对象添加 `TetrisSlotView`。Inspector 中配置：

- `inventorySlotType`：槽位接受的类型。
- `activeUIImage`：空槽显示图，可选。

`InventorySaveLoadService.inventorySlots` 中按固定顺序填入所有槽位。`slotIndex` 会在运行时由数组顺序写入 VM 和存档。

### 4.5 配置网格尺寸

`TetrisGridView` Inspector 中配置：

- `_gridSizeWidth`
- `_gridSizeHeight`
- `_localGridUnitSizeWidth`
- `_localGridUnitSizeHeight`

运行时 `TetrisGridView.Bind()` 会调用：

```csharp
viewModel.ApplyConfig(width, height, unitW, unitH);
```

并把 `RectTransform.sizeDelta` 绑定为：

```csharp
new Vector2(width * unitW, height * unitH)
```

## 5. 创建新物品

推荐使用编辑器窗口：

```text
Unity 菜单 -> CTIS -> Data Editor
```

编辑器支持：

- Items：维护 `ItemDataList_SO` 中的物品。
- Shapes：维护 `TetrisItemPointSet_SO` 中的形状点集。
- Config：维护 `InventoryPlacementConfig_SO`。
- 本地化表：使用 `ItemStrings`。

手动创建时流程如下：

1. 在 `TetrisPieceShape` 枚举中确认或新增形状名。
2. 在 `TetrisItemPointSet_SO.TetrisPieceShapeList` 中添加对应下标的 `PointSet`。
3. 在 `ItemDataList_SO.itemDetailsList` 添加 `ItemDetails`。
4. 设置 `itemID`，确保唯一。
5. 设置 `xWidth`、`yHeight`，它们必须能包住形状点集旋转前的外接矩形。
6. 设置 `tetrisPieceShape`、`itemIcon`、`inventorySlotType`、`itemRarity`、`maxStack`。
7. 如果物品是容器，设置 `gridUIPrefab`。
8. 如果使用本地化，补充 `localizedName` 和 `localizedDescription`。

## 6. 创建可嵌套容器物品

容器物品是“物品本身拥有一个或多个内部网格”。

配置步骤：

1. 给物品的 `ItemDetails.gridUIPrefab` 指向一个网格面板预制体。
2. 这个预制体内部放置一个或多个 `TetrisGridView`。
3. `TetrisItemView.InitializeGridPanel()` 会实例化这个预制体。
4. 每个内部 `TetrisGridView` 会通过 `TetrisItemVM.GetOrCreateGridVM(index)` 创建 VM。
5. 内部网格 GUID 格式为：

```text
{父物品 itemGuid}:{grid index}
```

系统会阻止把物品放进自己的内部容器，防止循环嵌套。规则在：

- `InventoryPlacementConfig_SO.blockSelfOwnedContainer`
- `TetrisUtilities.InventoryLogicHelper.IsPlacingIntoSelfOwnedContainer`
- `InventoryTreeCache.IsDescendantContainer`

## 7. 运行时创建和放置物品

如果你要通过代码生成一个物品并放入网格，核心流程是：

```csharp
using Cholopol.TIS;
using Cholopol.TIS.MVVM.ViewModels;

ItemDetails details = InventoryManager.Instance.itemDataList_SO.GetItemDetailsByID(itemId);

var data = new TetrisItemPersistentData
{
    itemID = details.itemID,
    itemGuid = System.Guid.NewGuid().ToString(),
    orginPosition = new Vector2Int(x, y),
    direction = details.dir,
    stack = 1,
    isOnSlot = false,
    persistentGridGuid = targetGrid.GridGuid,
    gridPIndex = targetGrid.GridPIndex,
};

TetrisItemVM itemVm = TetrisItemFactory.GetOrCreateVM(details, data, targetGrid);

bool placed = targetGrid.TryPlaceTetrisItem(itemVm, x, y);
```

注意：

- `targetGrid` 是 `TetrisGridVM`。
- `TryPlaceTetrisItem` 成功后会调用 `SetItemData()`，写入 `InventoryData_SO`。
- 如果你绕过 VM 直接改 `InventoryData_SO`，需要重新构建缓存和刷新 UI。

## 8. 放置规则和高亮

规则配置定义在：

- `Runtime/Core/Data/InventoryPlacementConfig_SO.cs`

可以创建配置资产：

```text
Create -> CTIS -> Placement Config
```

可配置规则：

- `blockSelfOwnedContainer`：禁止放进自己或自己的子容器。
- `blockOutOfBounds`：禁止越界。
- `blockSlotOccupied`：禁止放进已有物品的槽位。
- `blockSlotTypeMismatch`：禁止槽位类型不匹配。

可配置颜色：

- `ValidEmpty`：空位可放置。
- `Invalid`：不可放置。
- `CanStack`：可堆叠。
- `CanQuickExchange`：可快速交换。
- `invalidReasonColors`：按阻止原因覆盖颜色。

高亮逻辑在 `InventoryHighlight.UpdateShapeHighlightMVVM()` 中：

1. 根据 Ghost 的形状点集计算覆盖格。
2. 调用 `IInventoryService.CanPlace` 或 `InventoryPlacementConfig_SO.EvaluateActive`。
3. 根据目标格是否已有物品、是否可堆叠、是否可快速交换设置颜色。

## 9. 堆叠逻辑

堆叠判断在：

- `TetrisGridVM.TryPlaceTetrisItem`
- `TetrisGridVM.PlaceOnOverlapItem`
- `TetrisUtilities.InventoryLogicHelper.TryMergeStack`

触发条件：

- 拖到已有物品上。
- 两个物品 `itemID` 相同。
- `maxStack > 0`。
- 目标物品当前数量未满。

合并后：

- 目标物品 `CurrentStack` 增加。
- 来源物品 `CurrentStack` 减少。
- 来源物品数量归零时，从原网格、存档数据、UI 工厂注册表中移除。

## 10. 快速交换逻辑

快速交换位于 `TetrisGridVM.CanQuickExchange()` 和 `TetrisGridVM.TryQuickExchange()`。

它允许拖拽物品覆盖一个或多个目标物品，并尝试把被覆盖物品放回拖拽物品原来的区域。

成功条件大致是：

- Ghost 覆盖区域在目标网格边界内。
- 覆盖到的目标物品必须被完整覆盖，不能只覆盖一部分。
- 被覆盖物品能按映射或重新搜索放回原区域。
- 目标区域最终能放下拖拽物品。

这个功能复杂，扩展时建议先写小范围测试或在示例场景中验证多种形状组合。

## 11. 存档系统

主要类：

- `InventorySaveLoadService`
- `InventoryData_SO`
- `SaveLoadManager`
- `InventoryTreeCache`

数据流：

1. 物品放置、移动、旋转后，`TetrisItemVM.SetItemData()` 更新 `InventoryData_SO.inventoryItemList`。
2. `InventorySaveLoadService.GenerateSaveData()` 把 `InventoryData_SO` 包装成 `GameSaveData`。
3. `SaveLoadManager.Save(index)` 保存到指定存档槽。
4. 加载时 `InventorySaveLoadService.RestoreData()` 恢复 `InventoryData_SO`。
5. `BuildRuntimeCache()` 按 `persistentGridGuid` / `parentItemGuid:gridPIndex` 重建容器树。
6. UI 激活后，`InstantiateInventoryItemUICoroutine()` 延迟恢复物品 UI。

关闭背包时：

- `InventoryManager` 发布 `RecycleInventoryItemUI`。
- `InventorySaveLoadService` 回收物品 View，保留 VM/数据缓存。

打开背包时：

- `InventoryManager` 发布 `InstantiateInventoryItemUI`。
- `InventorySaveLoadService` 在预留 UI 激活后生成物品 View。

## 12. 常见扩展点

### 12.1 新增放置限制

优先扩展：

- `InventoryPlacementConfig_SO.Evaluate`
- 或 `InventoryService.CanPlace`

适合放在这里的规则：

- 背包重量上限。
- 物品类型白名单/黑名单。
- 特定网格只允许消耗品。
- 任务物品不可放入安全箱。

建议把“规则是否启用”和“规则参数”做成 `ScriptableObject` 配置，避免硬编码到 View。

### 12.2 新增物品属性

静态属性添加到 `ItemDetails`：

```csharp
public int durability;
public string itemCategory;
```

动态属性添加到 `TetrisItemPersistentData` 或 `CustomData`：

```csharp
public int currentDurability;
```

然后在 `TetrisItemVM` 中暴露属性，最后在 `TetrisItemView` 或信息面板绑定显示。

### 12.3 新增槽位类型

步骤：

1. 在 `InventorySlotType` 枚举中添加新类型。
2. 在物品 `ItemDetails.inventorySlotType` 中设置该类型。
3. 在对应 `TetrisSlotView.inventorySlotType` 中设置同类型。
4. 如有特殊规则，在 `InventoryPlacementConfig_SO` 或 `InventoryService` 中追加判断。

### 12.4 新增形状

步骤：

1. 在 `TetrisPieceShape` 枚举末尾添加新形状。
2. 在 `TetrisItemPointSet_SO.TetrisPieceShapeList` 末尾添加同顺序 `PointSet`。
3. 设置物品 `ItemDetails.tetrisPieceShape`。
4. 设置正确的 `xWidth` / `yHeight`。
5. 在示例场景测试四个方向旋转。

不要在已有项目中间插入枚举值，否则老数据的枚举下标可能对应到错误形状。

### 12.5 替换物品 UI 预制体

通用物品 UI 预制体由 `CTISPrefabConfig.generalTetrisItemPrefab` 管理，并通过 Addressables 加载。

替换时确保新预制体至少具备：

- `TetrisItemView`
- 可用于显示图标的 `Image`，或让 `TetrisItemView.Initialize()` 自动创建。
- 用于接收拖拽的内容 Image，通常需要 `SpriteMeshRaycastFilter`。

如果要增加文本、品质条、耐久度条，建议在 `TetrisItemVM` 暴露可绑定属性，再在 `TetrisItemView.Bind()` 中添加绑定。

### 12.6 替换右键菜单或信息面板

相关类：

- `RightClickMenuPanel`
- `ItemInformationPanel`
- `FloatingTetrisGridWindow`

物品详情面板可以用：

```csharp
ItemInformationPanel.OpenAsync(itemDetails);
```

容器浮窗可以用：

```csharp
FloatingTetrisGridWindow.OpenAsync(itemVm);
```

如果扩展右键功能，建议让菜单只负责收集上下文和触发命令，实际物品操作放到 Service 或 VM。

## 13. 调试建议

### 13.1 放不进去

检查顺序：

1. 目标是否是 `TetrisGridView` 或 `TetrisSlotView`。
2. `InventoryManager.selectedTetrisItemGridVM` 是否正确更新。
3. 目标网格 `GridSizeWidth` / `GridSizeHeight` 是否正确。
4. 物品 `xWidth` / `yHeight` 是否覆盖了形状点集。
5. `TetrisItemOccupiedCells` 是否已有占用。
6. `InventoryPlacementConfig_SO` 是否阻止了放置。
7. 是否触发了“放进自己容器”的保护。

### 13.2 物品显示位置错位

重点检查：

- Canvas `scaleFactor`。
- `TetrisGridView._localGridUnitSizeWidth` / `_localGridUnitSizeHeight`。
- 物品 `Width` / `Height`。
- `RotationOffset`。
- `TetrisGridVM.CalculatePositionOnGrid()`。

### 13.3 存档后物品丢失

重点检查：

- 物品是否有稳定 `itemGuid`。
- 所在持久化网格是否有 `DataGUID`。
- `InventorySaveLoadService.inventoryPersistentGrids` 是否包含目标网格。
- `InventorySaveLoadService.inventorySlots` 是否包含目标槽位。
- `InventoryData_SO.inventoryItemList` 中是否存在对应记录。

### 13.4 内部背包不恢复

重点检查：

- 父物品 `itemGuid` 是否稳定。
- 内部网格 GUID 是否符合 `{itemGuid}:{index}`。
- `gridPIndex` 是否正确。
- 父物品的 `gridUIPrefab` 是否包含 `TetrisGridView`。
- `InventoryTreeCache` 是否在加载时收到对应容器。

## 14. 推荐修改原则

- 规则改 Service / Config，不要写进 View。
- UI 表现改 View，不要改 VM 的核心放置逻辑。
- 物品静态配置放 `ItemDetails`，物品实例状态放 `TetrisItemPersistentData`。
- 已发布项目不要随意改 `DataGUID`、`itemID`、枚举顺序。
- 新增形状后必须测试四个方向旋转、边界放置、拖拽高亮。
- 新增存档字段后要考虑旧存档默认值。

## 15. 常用文件索引

| 功能 | 文件 |
| --- | --- |
| 全局交互、Ghost 旋转、高亮调度 | `Runtime/Managers/InventoryManager.cs` |
| 物品静态/动态数据结构 | `Runtime/Core/Data/DataCollections.cs` |
| 枚举 | `Runtime/Core/Data/Enums.cs` |
| 物品数据库 | `Runtime/Core/Data/ItemDataList_SO.cs` |
| 形状数据库 | `Runtime/Core/Data/TetrisItemPointSet_SO.cs` |
| 放置规则配置 | `Runtime/Core/Data/InventoryPlacementConfig_SO.cs` |
| 网格 VM | `Runtime/MVVM/VM/TetrisGridVM.cs` |
| 物品 VM | `Runtime/MVVM/VM/TetrisItemVM.cs` |
| 槽位 VM | `Runtime/MVVM/VM/TetrisSlotVM.cs` |
| Ghost VM | `Runtime/MVVM/VM/TetrisItemGhostVM.cs` |
| 网格 View | `Runtime/MVVM/V/TetrisGridView.cs` |
| 物品 View | `Runtime/MVVM/V/TetrisItemView.cs` |
| 槽位 View | `Runtime/MVVM/V/TetrisSlotView.cs` |
| 高亮 | `Runtime/Core/Grid/InventoryHighlight.cs` |
| 持久化网格描述 | `Runtime/Core/Grid/InventoryGridDescriptor.cs` |
| 物品/网格工厂 | `Runtime/Core/Factory` |
| 存档服务 | `Runtime/SaveLoad/InventorySaveLoadService.cs` |
| 运行时容器树缓存 | `Runtime/MVVM/Services/InventoryTreeCache.cs` |
| 编辑器窗口 | `Editor/UI Builder/ChosTISDataEditor.cs` |
| 预制体配置 | `Config/CTISPrefabConfig.cs` |

