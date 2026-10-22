# 编写附属工作站

[English](../en/station-kit.md)

本页是“附属自建工作站”的接线指南：用 CraftEngine 的方块与方块实体，复用 FarmersDelight 的容器 GUI、配方书、
配方编辑器、调度与保护，而不是重新实现一套。三个参考实现：Brewin' And Chewin' 的酒桶（`KegManager`、
`KegRecipeType`）、Crabber's Delight 的蟹笼（`CrabTrapBlockEntityController`）、Barbeque's Delight 的烤架
（`StationBlockEntity`）。

本页没有新增 api 类——这套“SDK”是 `api.gui`、`api.recipe`、`api.block`、`api.scheduler`、
`api.util.ProtectionCompat` 与 CE 方块/方块实体类型的固定组合顺序。

## 1. 方块与方块实体

* 在自己的 CraftEngine 包里声明方块与物品，按需写 `settings.tags` 以加入 `farmersdelight:*` / `c:*` 标签
  （见[标签](tags.md)）。
* 方块实现 CE 的 `EntityBlock` 行为，由它创建方块实体控制器；库存状态放在控制器里，用稳定的 custom-data key
  在 `saveCustomData` / `loadCustomData` 中持久化，保证重启后恢复。
* 在 `onLoad` 注册行为工厂：CE 在插件 `onEnable` 之前解析内容，晚注册的行为只对 `/ce reload` 生效。
* 世界访问只放在拥有该方块的 region 上：交互、破坏/放置事件或 `runAtLocation` 内。

## 2. Tick

用 `FarmersDelightApi` 的调度器，而不是 Bukkit 调度器：

```java
FarmersDelightApi api = FarmersDelightApi.get();
ApiTask task = api.runRepeating(() -> {
    // 轻量扫描：只给已加载且不空闲的工作站安排真正的工作
    api.runAtLocation(location, () -> tickStation());
}, 1L, 4L);
```

要点：

* 工作站有工作（投入物品、开始一批）时启动循环，空闲时取消；不要永久 tick 每个已放置的工作站。
* `runRepeating` 在 Folia 上跑全局 region、Paper 上跑主线程；所有世界/库存操作都经
  `runAtLocation` / `runLaterAtLocation` 回到所属 region。
* 每 pass 控制预算：限制一次处理的站数，绝不每 tick 遍历全部已加载方块。
* `onDisable` 与方块实体卸载时取消任务。

## 3. 容器 GUI

`api.gui` 就是内置工作站用的引擎：

* 用 `GuiLayout`（或 `GuiLayouts`）把格子组（`input`、`output`、`container`）映射到扁平 store；
  `ContainerGuiKernel` 负责槽位与 store 的对应并拒绝过期回写。
* 实现 `ContainerGuiKernel.Controller`：`layout()`、`slotCount()`、`stored()`、`store()`、`storeLock()`、
  `markDirty()`；保留前先 clone，空槽存 `null`。
* `GuiItems` 生成装饰物，`GuiWriteBack` 提交一次被接受的点击。
* 只处理该工作站自己的 `InventoryClickEvent`/`InventoryDragEvent`，确认仍是玩家当前打开的界面，并在工作站
  线程上执行提交。

## 4. 配方与配方书

* 实现 `RecipeType` + `ViewableRecipe`，用 `FarmersDelightApi.registerRecipeType` 注册；配方书无需知道你的内部
  格式，`editor()`、`listLayout()`/`detailLayout()` 把其余部件接上。
* 复用 FD 的原料语义时，适配成 `RecipeIngredient`，让 `IngredientMatching`/`RecipeFiller` 继续可用；否则保留
  自己的匹配器，只把配方 api 用于展示。
* 配方文件用 `RecipeEditorFiles` / `AddonRecipeFiles` 读写；`registerCookingPotRecipe` /
  `registerCuttingBoardRecipe` 只作用于内置工作站。
* 用 `openRecipeBook(player, typeId, filler)` 打开配方书。

## 5. 编辑器

`RecipeType.editor()` 返回 `RecipeEditor`；用 `EditableRecipe` 实现，并通过 `RecipeEditorFiles` 持久化。保存
昂贵时改实现 `AsyncRecipeEditor`：后台保存，再经玩家/方块调度器回到工作站 region 应用结果。契约见
[RecipeEditor](recipe-editor.md)。

## 6. 保护、标签、事件与成就

* 保护：调用 `api.util.ProtectionCompat.canUse` / `canBuild`，用自己的 WorldGuard flag（附属在 `onLoad` 注册，
  `paper-plugin.yml` 中把 `WorldGuard` 列为可选依赖）。没有保护插件时按内置工作站的权限约定退化为
  `PermissionChecker`。
* 用 `registerAddonBlockNamespace("myaddon")` 注册命名空间，方块状态占用报告会统计你的工作站。
* 标签：包内 `settings.tags`；只有运行期映射才用 `registerCommonTags`。
* 事件：内置工作站事件（`FarmersDelightStationPlacedEvent` / `BrokenEvent`）只覆盖 FD 的四种工作站；自建
  工作站需要生命周期钩子时自定义事件，线程措辞参考[事件](events.md)。
* 成就：通过 `api.advancement` 注册成就树，或复用 FD 的触发注册
  （`registerAdvancementItemTrigger` / `registerAdvancementConsumeTrigger`）。

## 7. 骨架

```java
public final class MyStationBehavior extends EntityBlock implements WorldlyContainerHolder {
    @Override
    public BlockEntityController createBlockEntityController(CEWorld world, ImmutableBlockState state) {
        return new MyStationController(world, state, this); // 持有库存与数据
    }

    // 交互：保护检查 -> 打开 GUI 或投入；所有写入都走控制器。
    // 控制器自己用 api.runRepeating / runAtLocation 安排循环。
}
```

骨架刻意很小：方块实体控制器管状态，调度器管时序，容器 GUI / 配方 / 编辑器都是上面的 api 类型，而不是从
FarmersDelight 复制代码。

## 8. 不要照抄的部分

* 不要把 `CookingPotBlockEntity` 或 `CuttingBoardBlockBehavior` 当起点：它们带着内置工作站的进度、漏斗、展示、
  配方发现等逻辑，自建工作站不需要。
* 不要每 tick 对全部已加载工作站 `world.getBlockAt`；在管理器里跟踪活跃站，空闲就取消循环。
* 不要在控制器锁之外保存 `ItemStack[]`；容器 GUI 与漏斗路径共享同一把锁。

## 相关页面

* [方块与工作站](blocks-and-stations.md)
* [容器 GUI](container-gui.md)
* [RecipeType 与 ViewableRecipe](recipe-type.md)
* [RecipeEditor](recipe-editor.md)
* [调度与 ApiTask](scheduling.md)
* [标签](tags.md)
* [事件](events.md)
