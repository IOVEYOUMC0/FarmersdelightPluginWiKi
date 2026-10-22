# Building an addon station

[简体中文](../zh-cn/station-kit.md)

This page is the wiring guide for an addon-owned station: a CraftEngine block and block entity that reuses
FarmersDelight's container GUI, recipe book, editor, scheduling and protection instead of re-implementing them.
The three reference implementations are Brewin' And Chewin's keg (`KegManager`, `KegRecipeType`), Crabber's
Delight's crab trap (`CrabTrapBlockEntityController`) and Barbeque's Delight's grill (`StationBlockEntity`).

Nothing on this page is a new api class — the kit is the composition of `api.gui`, `api.recipe`, `api.block`,
`api.scheduler`, `api.util.ProtectionCompat` and the CE block/block-entity types. The point of the page is that
those pieces fit together in a fixed order.

## 1. Block and block entity

* Declare the block and its item in your own CraftEngine pack, including `settings.tags` for any
  `farmersdelight:*` / `c:*` membership it needs (see [Tags](tags.md)).
* Make the block an `EntityBlock` whose behavior creates a block-entity controller. The controller is where your
  inventory state lives; persist it in `saveCustomData` / `loadCustomData` with a stable custom-data key so a
  server restart restores the station.
* Register the behavior factory from `onLoad`: CraftEngine parses content before plugin `onEnable`, so a factory
  registered later only applies to `/ce reload`.
* Keep world access on the region that owns the block. Read/write block state only from a handler that already
  runs there (interaction, block break/place, `runAtLocation`).

## 2. Ticking

Use `FarmersDelightApi`'s scheduler instead of Bukkit's:

```java
FarmersDelightApi api = FarmersDelightApi.get();
ApiTask task = api.runRepeating(() -> {
    // Cheap sweep: only enqueue real work for stations that are loaded and not idle.
    api.runAtLocation(location, () -> tickStation());
}, 1L, 4L);
```

Rules that keep this cheap and Folia-safe:

* Start the loop when the station has work (an item inserted, a player interaction that starts a batch) and
  cancel it when the station falls idle; do not tick every placed station forever.
* `runRepeating` runs on the global region (Folia) or the main thread; all world/inventory work goes through
  `runAtLocation` / `runLaterAtLocation` so it executes on the owning region.
* Keep one budgeted pass per station: cap how many stations one pass touches, and never loop over all loaded
  blocks per tick.
* Cancel the task in `onDisable` and when the block entity is unloaded.

## 3. Container GUI

`api.gui` is the same engine the built-in stations use:

* Build a `GuiLayout` (or use `GuiLayouts`) that maps cell groups (`input`, `output`, `container`) to a flat
  store; `ContainerGuiKernel` maps slots to that store and rejects stale write-backs.
* Implement `ContainerGuiKernel.Controller` over your block entity: `layout()`, `slotCount()`, `stored()`,
  `store()`, `storeLock()`, `markDirty()`; clone what you keep and store `null` for empty.
* `GuiItems` builds decoration (background/placeholder), `GuiWriteBack` commits an accepted click.
* Handle `InventoryClickEvent`/`InventoryDragEvent` only for the station's own inventory, verify it is still the
  player's open inventory, and run the kernel commit on the station's thread.

## 4. Recipes and the recipe book

* Implement `RecipeType` + `ViewableRecipe` and register it with `FarmersDelightApi.registerRecipeType`. The book
  renders your recipes without knowing your internal format; `RecipeType.editor()` and
  `RecipeType.listLayout()`/`detailLayout()` hook the rest of the kit in.
* If your station reuses FD's ingredient semantics, adapt to `RecipeIngredient` so `IngredientMatching` and
  `RecipeFiller` keep working; otherwise keep your own matcher and use the recipe api only for display.
* Recipes can be edited through `RecipeEditorFiles` / `AddonRecipeFiles`; runtime registration through
  `registerCookingPotRecipe`/`registerCuttingBoardRecipe` only targets the built-in stations.
* Open the book with `openRecipeBook(player, typeId, filler)`.

## 5. Editor

`RecipeType.editor()` returns a `RecipeEditor`; implement it with `EditableRecipe` records and persist through
`RecipeEditorFiles`. When saving is expensive, implement `AsyncRecipeEditor` instead: save off-thread and apply
the result back on the station's region thread through the player/block scheduler. The editor contract and the
async variant are documented in [RecipeEditor](recipe-editor.md).

## 6. Protection, tags, events and advancements

* Protection: call `api.util.ProtectionCompat.canUse` / `canBuild` with your own `Feature` (WorldGuard flags are
  registered by the addon; see the addon's `paper-plugin.yml` for `WorldGuard` as a soft dependency). For a
  standalone permission check fall back to `PermissionChecker` conventions used by the built-in stations.
* Register the block namespace with `registerAddonBlockNamespace("myaddon")` so the block-state usage report
  counts your stations.
* Tags: `settings.tags` in your pack, `registerCommonTags` only for runtime mappings.
* Events: the built-in station events (`FarmersDelightStationPlacedEvent` / `BrokenEvent`) cover FD's four
  stations only. If your station needs its own lifecycle hooks, define your own events; reuse the threading
  wording from [Events](events.md).
* Advancements: register a tab through `api.advancement` or feed FD's advancement triggers
  (`registerAdvancementItemTrigger` / `registerAdvancementConsumeTrigger`).

## 7. Skeleton

```java
public final class MyStationBehavior extends EntityBlock implements WorldlyContainerHolder {
    @Override
    public BlockEntityController createBlockEntityController(CEWorld world, ImmutableBlockState state) {
        return new MyStationController(world, state, this); // owns inventory + data
    }

    // Interaction: protection -> open GUI or insert; every write goes through the controller.
    // The controller schedules its own loop with api.runRepeating / runAtLocation.
}
```

The skeleton is deliberately small: the block-entity controller owns state, the scheduler owns timing, and the
container GUI/reception/editor pieces are the api types above rather than code copied from FarmersDelight.

## 8. What not to copy

* Do not copy `CookingPotBlockEntity` or `CuttingBoardBlockBehavior` as a starting point; they carry built-in
  station logic (progress, hopper, display, recipe discovery) that an addon station does not need.
* Do not poll `world.getBlockAt` for every loaded station each tick; track active stations in the manager and
  cancel the loop on idle.
* Do not store `ItemStack[]` outside the controller's lock; the container GUI and hopper paths share it.

## Related pages

* [Blocks and stations](blocks-and-stations.md)
* [Container GUI](container-gui.md)
* [RecipeType and ViewableRecipe](recipe-type.md)
* [RecipeEditor](recipe-editor.md)
* [Scheduling and ApiTask](scheduling.md)
* [Tags](tags.md)
* [Events](events.md)
