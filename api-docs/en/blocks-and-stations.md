
[简体中文](../zh-cn/blocks-and-stations.md)
# Blocks and Stations

Package: `com.huidu.farmersdelight.api.block`

This package answers two questions for an addon: *what is this block*, and *what does this station
currently hold*. Every parameter and return type is a Bukkit or `java` type (plus the snapshot records
themselves), so an addon can use it without adding a CraftEngine dependency.

Types covered here:

| Type | Kind | `@ApiStatus` |
| --- | --- | --- |
| `FarmersDelightBlocks` | static facade, private constructor | `@Experimental`, `@NonExtendable` |
| `FarmersDelightStation` | enum, 4 constants | `@Experimental` |
| `CookingPotSnapshot` | record | `@Experimental` |
| `CuttingBoardSnapshot` | record | `@Experimental` |
| `SkilletSnapshot` | record | `@Experimental` |
| `StoveSnapshot` | record | `@Experimental` |
| `CraftEngineBlockAccess` | static facade, private constructor | — |

`@ApiStatus.Experimental` means the signatures may still change between FarmersDelight releases — pin
the version you compile against and re-check on upgrade. `@ApiStatus.NonExtendable` on
`FarmersDelightBlocks` is doubly enforced: the class is `final` with a private constructor.

## Never identify a station by Material

FarmersDelight stations are CraftEngine custom blocks.
CraftEngine reports a configurable *disguise* material through Bukkit — by default the same one for
every custom block — so this identifies nothing:

```java
// WRONG. Matches every custom block on the server, or none, depending on a config value
// the server owner can edit at any time.
if (block.getType() == Material.BRICKS) { /* "a cooking pot" */ }
```

Part of why this API exists is to make the correct call easy:

```java
import com.huidu.farmersdelight.api.block.FarmersDelightBlocks;
import com.huidu.farmersdelight.api.block.FarmersDelightStation;

FarmersDelightStation station = FarmersDelightBlocks.stationOf(block);
if (station == FarmersDelightStation.COOKING_POT) {
    // definitely a cooking pot
}
```

`stationOf` resolves the real CraftEngine block state and detects the station by the **block behavior**
attached to it, not by its block id. A server that re-skins a station under its own id, or an addon
that reuses a station behavior on its own block, is still recognised.

### The second half of the red line: getting an id right

The obvious-looking route from a CraftEngine block state to its id is a trap. The state's owner exposes
an `Optional` wrapper around a resource key; calling `toString` on that wrapper yields the wrapper's own
text form (an `Optional` rendering of a resource key), never a plain `namespace:path` id. Comparing that
against `"farmersdelight:cooking_pot"` is *always* false, so the guarded code becomes dead and nothing
logs an error. The id has to come from the resource key's location.

`blockIdOf` and `stationIdOf` already do that. Use them instead of walking the CraftEngine state
yourself.

```java
String id = FarmersDelightBlocks.blockIdOf(block);   // "farmersdelight:cooking_pot", "brewinandchewin:keg", or null
```

## Threading (Folia / Luminol)

Every method in `FarmersDelightBlocks` reads world state. The caller must already be on the region
thread that owns `block`:

* inside a listener for an event at that block, or
* inside a task scheduled with `FarmersDelightApi.get().runAtLocation(block.getLocation(), ...)`.

Calling from another region's thread, from the global region, or from an async task reaches Paper's
thread check in `CraftWorld` and **throws** (the moonrise tick-thread assertion). It surfaces as an
exception inside your listener, not as a `null` return.

On a single-threaded Paper server there is one region, so any main-thread call works — which is exactly
why this is easy to miss until someone runs your addon on Folia.

These methods never schedule for you. A snapshot has to be returned synchronously, and hopping regions
would hand you data from a different tick.

```java
import com.huidu.farmersdelight.api.FarmersDelightApi;

FarmersDelightApi.get().runAtLocation(block.getLocation(), () -> {
    CookingPotSnapshot pot = FarmersDelightBlocks.cookingPot(block);
    if (pot != null && pot.cooking()) {
        // ...
    }
});
```

## FarmersDelightStation

```java
public enum FarmersDelightStation {
    COOKING_POT,    // defaultBlockId() = "farmersdelight:cooking_pot"
    CUTTING_BOARD,  // defaultBlockId() = "farmersdelight:cutting_board"
    SKILLET,        // defaultBlockId() = "farmersdelight:skillet"
    STOVE;          // defaultBlockId() = "farmersdelight:stove"

    public String defaultBlockId();
}
```

`defaultBlockId()` is the id FarmersDelight *ships* the station under. It is **not** what
`FarmersDelightBlocks.stationIdOf` returns — that returns the id of the block actually in the world,
which may differ on a server that re-skins a station. Compare block ids only when you specifically mean
"the stock station".

## FarmersDelightBlocks

```java
public static String                blockIdOf(Block block);
public static String                stationIdOf(Block block);
public static boolean               isStation(Block block);
public static FarmersDelightStation stationOf(Block block);

public static CookingPotSnapshot    cookingPot(Block block);
public static CuttingBoardSnapshot  cuttingBoard(Block block);
public static SkilletSnapshot       skillet(Block block);
public static StoveSnapshot         stove(Block block);
```

`blockIdOf` — the CraftEngine block id of `block` (e.g. `"farmersdelight:cooking_pot"`, or an addon's
`"brewinandchewin:keg"`), or `null` when the block is not a CraftEngine custom block at all. Works for
any custom block, not only FarmersDelight's.

`stationIdOf` — the block id when `block` is one of the four stations, otherwise `null`. Use `stationOf`
when you want to branch on *which* station it is.

`isStation` — true when `block` is any of the four stations.

`stationOf` — which station `block` is, or `null` when it is not one.

All four accept a `null` block and return `null` / `false` for it.

### The snapshot getters

Each getter returns `null` when the block is not that station. Beyond that, each has its own "not
tracked" case:

| Getter | Additionally returns null when |
| --- | --- |
| `cookingPot` | no block entity is loaded — an unloaded chunk, or a pot placed this tick that has not been initialised |
| `cuttingBoard` | no block entity is loaded. An **empty board still returns a snapshot**, with a `null` `storedItem` |
| `skillet` | the skillet is not tracked — nothing has ever been placed in or on it |
| `stove` | the stove is not tracked — no food has ever been put on it |

They also return `null` when FarmersDelight itself is not loaded or not enabled, so an addon does not
have to guard the call separately.

## What a snapshot is, and what it is safe for

A snapshot is a **copy taken at the instant of the call**. The station keeps ticking afterwards. Treat
the values as a reading, not as a handle.

Safe uses: displaying station contents in a GUI or hologram, gating logic ("does this pot already hold
a meal?"), analytics, debug commands, deciding whether to fire your own event.

Not possible through this API: writing. Every `ItemStack` is cloned on the way into the record *and*
again on every accessor call, so mutating what you were handed can never reach the station's live
inventory, and the station can never hand you a live stack. Lists are unmodifiable. `location()` returns
a clone too.

```java
CookingPotSnapshot pot = FarmersDelightBlocks.cookingPot(block);
pot.ingredients().get(0).setAmount(64);   // compiles, but changes nothing anywhere
```

There is no supported mutation route. If you need to change what a station holds, drive it through
normal gameplay (hoppers, player interaction) or use the station's own event surface.

### CookingPotSnapshot

```java
public record CookingPotSnapshot(Location location,
                                 List<ItemStack> ingredients,
                                 ItemStack container,
                                 ItemStack mealDisplay,
                                 ItemStack output,
                                 String recipeId,
                                 int progressTicks,
                                 int cookTimeTicks,
                                 int remainingTicks,
                                 boolean heated) {
    public boolean cooking();
    public double progressFraction();
}
```

* `location` — the pot's block location, block-aligned corner.
* `ingredients` — the ingredient slots in layout order. A `null` entry means an empty slot, and null
  entries are preserved so slot indexes stay meaningful.
* `container` — the bowl/bottle slot's contents, or `null`.
* `mealDisplay` — the first item in the pot's **output** slots: the finished meal currently shown in
  the pot. `null` when the pot holds none.
* `output` — the first item in the pot's **pending-output** slots: a just-cooked meal that has not been
  moved on yet. `null` when that slot is empty.
* `recipeId` — the recipe currently being cooked, or `null` when the pot is idle. Sourced from the
  matched `CookingPotRecipe`'s id.
* `progressTicks` / `cookTimeTicks` — ticks accumulated and ticks needed in total. `cookTimeTicks` is 0
  when idle and no duration is set.
* `remainingTicks` — `cookTimeTicks - progressTicks`, floored at 0.
* `heated` — whether a configured heat source (or a conductor over one) is under the pot.
* `cooking()` — true when `recipeId != null`.
* `progressFraction()` — 0..1, and 0 when `cookTimeTicks <= 0`.

Note the `mealDisplay` / `output` split above: despite the names, `mealDisplay` reads the output slot
and `output` reads the pending-output slot.

### CuttingBoardSnapshot

```java
public record CuttingBoardSnapshot(Location location, ItemStack storedItem, boolean carved) {
    public boolean occupied();
}
```

A cutting board holds at most one stack. `carved` is whether the stored item is displayed in the
"carved" (tool) pose rather than flat. `occupied()` is true when `storedItem != null`.

### SkilletSnapshot

```java
public record SkilletSnapshot(Location location,
                              ItemStack storedItem,
                              ItemStack skilletItem,
                              String recipeId,
                              int progressTicks,
                              int cookTimeTicks,
                              int remainingTicks,
                              boolean heated,
                              int fireAspectLevel) {
    public boolean cooking();
    public double progressFraction();
}
```

* `storedItem` — the food currently in the pan, or `null`.
* `skilletItem` — the skillet item the block was placed from, carrying its enchantments, or `null`.
* `recipeId` — the campfire recipe key being cooked (a Bukkit `NamespacedKey` rendered as a string), or
  `null` when nothing matches.
* `heated` — a configured heat source (or conductor) under the skillet.
* `fireAspectLevel` — the Fire Aspect level on the skillet item, which shortens the cook time.
* `cooking()` — true when a recipe matches. Progress only advances while `heated`.

### StoveSnapshot

```java
public record StoveSnapshot(Location location,
                            List<ItemStack> items,
                            List<Integer> progressTicks,
                            List<Integer> cookTimeTicks,
                            boolean lit,
                            boolean blockedAbove) {
    public int occupiedSlots();
    public double progressFraction(int slot);
}
```

A stove grills several items at once. The three lists are **parallel and always the same length** — one
entry per grilling slot, indexed the way the stove tracks them. The stove has 6 slots, so all three
lists have 6 entries. A `null` entry in `items` means an empty slot.

* `lit` — whether the stove is burning. An unlit stove makes no progress.
* `blockedAbove` — whether a collision shape above the stove blocks the grilling area.
* `occupiedSlots()` — number of slots holding food.
* `progressFraction(int slot)` — 0..1 for that slot; 0 for an out-of-range slot or an unset duration
  (never throws).

```java
StoveSnapshot stove = FarmersDelightBlocks.stove(block);
if (stove != null && stove.lit() && !stove.blockedAbove()) {
    for (int slot = 0; slot < stove.items().size(); slot++) {
        ItemStack food = stove.items().get(slot);
        if (food != null) {
            player.sendMessage(food.getType() + " " + (int) (stove.progressFraction(slot) * 100) + "%");
        }
    }
}
```

A note on cost: every accessor clones. In a per-tick loop, call `items()` once into a local rather than
inside the loop condition.

## CraftEngineBlockAccess

Final, with no `@ApiStatus` annotation. The class exposes the CraftEngine block plumbing FarmersDelight uses
internally — resolving a CE world, turning a block entity back into a Bukkit world, reading a state's id or
facing, marking a chunk dirty — published so addons do not each rewrite it. Every method is null-tolerant and
returns `null` / `false` rather than throwing, because the callers are block behaviours and chunk listeners
where an exception aborts something much larger.

```java
public static CEWorld                getCEWorld(World world);
public static World                  getBukkitWorld(BlockEntity blockEntity);
public static World                  toWorld(Object levelHandle);
public static BlockPos               toBlockPos(Object posHandle);
public static void                   markDirty(BlockEntity blockEntity);
public static String                 blockId(ImmutableBlockState state);
public static BlockFace              facing(ImmutableBlockState state);
public static String                 property(ImmutableBlockState state, String propertyName);
public static BlockEntityController  blockEntityController(World world, int x, int y, int z);
```

The parameter and return types are CraftEngine's, so this part of the surface needs CraftEngine on your
compile classpath — which it already is if you write behavior classes.

`getCEWorld` is the one worth calling instead of the obvious alternative. Do not resolve the world yourself
through `BukkitWorldManager.instance().getWorld(uuid).ceWorld()`: before CraftEngine binds its blocks,
resolving a world deserializes saved chunk data against an unbound registry; a block left over from an
uninstalled pack then throws, and because chunk-load handlers are a common caller it throws for every chunk.
`getCEWorld` applies the readiness gate, tolerates both CraftEngine return shapes and logs the failure once
instead of propagating it. A `null` result means CraftEngine is not ready or the world's data cannot be
loaded — not that the world has no CE content.

`markDirty` flags the block entity's chunk as unsaved. CraftEngine serialises only chunks marked dirty, so
every state change that must survive a restart has to be followed by it. `blockId` reads a CE block state's
custom id (`"namespace:id"`, or `null` when it is not a custom block); `facing` defaults to `NORTH` when the
state has no facing property; `property` returns `null` when the state has no such property.

### `blockEntityController(World world, int x, int y, int z)`

The block-entity controller at a world position, or `null` when it cannot be read. It returns `null` in
exactly three cases:

* the world is unknown to CraftEngine — CraftEngine is not ready yet, or the world's data cannot be loaded
  (the condition `getCEWorld` screens for);
* the chunk is **not loaded**;
* the position holds no block entity.

Only an already loaded chunk is consulted, so this never loads one and never returns data a region would have
to be scheduled for. It is still a read of world state, so the region-thread rule above applies — and `null`
on an unloaded chunk means "not readable now", not "no station here". Load the chunk on its owning thread
first if that distinction matters.

## Related pages

* [Items](items.md)
* [Events](events.md)
* [The recipe package](recipes-overview.md)
