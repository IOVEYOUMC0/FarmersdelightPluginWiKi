
[简体中文](../zh-cn/debug-tools.md)
# Debug tools — `DebugToolExtension` and `DebugToolRegistry`

FarmersDelight has an admin command, `/fd debugtools`, for stress-testing: mass-place blocks, mass-activate
them so they actually tick, inspect live state, validate loaded recipes, and undo the placement. The two types in
`com.huidu.farmersdelight.api.util` let an addon plug its own block into that command instead of shipping a
debug CLI of its own.

The feature id is `debug-tools`.

Use `/fd stats profile 200 all` to measure feature timings. Replace `all` with `handheld`,
`handheld_display`, `cooking_pot`, `skillet` or `stove` to select one feature. Release builds support
this command too. See [performance troubleshooting](../../server-guide/en/troubleshooting.md) for
timing boundaries and interpretation. Debug builds use the `-debug.jar` suffix; install either the
release or debug jar, never both. `/fd perf` always aliases `/fd stats`.

Debug builds can create a small, local test load manually:

```text
/fd debugtools test cooking_pot 64 200
/fd debugtools test skillet 64 200
/fd debugtools test stove 64 200
/fd debugtools test all 128 600
```

The command places stations near the player, fills their test state, and starts a matching profile. It processes at most 16 positions per slice so the command does not occupy the server thread for one long operation. `count` is the number of positions and `ticks` is the profile duration (20-12000). Use `/fd debugtools undo` to remove the latest batch and `/fd debugtools stop` to stop an unfinished placement, activation, or handheld test. `all` covers cooking pots, placed skillets and stoves.

Use `/fd debugtools test handheld 200 200` for simulated computation load: count independent sessions, without held items, repeated right-clicking or nearby heat. It consumes no player items. The result reports updates, display builds, completions and computation time per simulated tick. It covers slot matching, progress, display copies and completion recipe checks; it excludes networking, experience events, live inventories and Folia region dispatch, and is not the MSPT of 200 real players. It uses the loaded beef campfire recipe time. `stop` reports the partial sample and ends it; logout cleans up. Use `/fd stats profile 200 handheld` and `handheld_display` for real handheld sessions.
Recommended nested commands:

```text
/fd debugtools place recipe cooking_pot farmersdelight:beef_stew 4
/fd debugtools place random all 16
```

Legacy `placeRecipe` and `placeRecipeRandom` aliases remain. Undo handles your placement batch; standalone activate has no separate undo and consumed or collected outputs are not restored. Stop ends pending batch work or simulation; it is not a full rollback.


The `recipe validate` action checks loaded cooking-pot and cutting-board recipes for empty structure,
missing results/containers, and tags that resolve to no vanilla, CraftEngine, or registered common-tag
members. Parse failures are still reported by the normal `recipe.load_failed` logger.

## Availability: this only runs on a debug build

`/fd debugtools` exists only in the debug build of FarmersDelight; on a normal release build the command class
is absent from the jar and the command is never registered. Test your integration on a debug build. On a
release build the registry is mostly **dormant**: `register` still works and nothing ever calls your `place`,
`activate` or `cleanupBeforeUndo`. `status` is the exception — it is invoked by the always-resident
`/fd stats addon` subcommand, which works even on a release build.

That is why registering unconditionally is safe, and why both FDAddonTemplate and BrewinAndChewin do exactly
that in `onEnable` without probing anything.

## `DebugToolRegistry`

```java
package com.huidu.farmersdelight.api.util;

@ApiStatus.NonExtendable
public final class DebugToolRegistry {

    public static void register(DebugToolExtension extension);
    public static void unregister(String name);
    @Nullable public static DebugToolExtension find(String name);
    public static Collection<String> registeredNames();
    public static Collection<DebugToolExtension> all();
}
```

A static, process-wide `ConcurrentHashMap` keyed by the lowercased extension name. Notes:

- `register` is idempotent — re-registering the same name replaces the previous extension. `null`
  extensions, and extensions whose `name()` is `null`, are ignored.
- `unregister(name)` must be called from your `onDisable`. The map is static and survives your plugin's
  classloader, so a stale entry keeps a torn-down manager reachable and will be invoked on the next
  `/fd debugtools` run.
- `find` returns `null` for an unknown or `null` name. `registeredNames()` and `all()` return unmodifiable
  views; `registeredNames()` is what feeds the command's tab-completion.

```java
@Override
public void onEnable() {
    // ...
    DebugToolRegistry.register(new ExampleDebugExtension(this));
}

@Override
public void onDisable() {
    DebugToolRegistry.unregister("example_block");
}
```

## `DebugToolExtension`

```java
package com.huidu.farmersdelight.api.util;

@ApiStatus.OverrideOnly
public interface DebugToolExtension {

    String name();

    int place(Player player, Location origin, int count, int spacing, int layers, UndoSink undo);

    @FunctionalInterface
    interface UndoSink {
        void capture(Location loc);
    }

    default int activateAt(Player player, Location location) { return 0; }

    default List<String> status(Player player) { return List.of(); }

    default void cleanupBeforeUndo(Location location) { }
}
```

`@ApiStatus.OverrideOnly` means the opposite of `NonExtendable`: you are expected to implement this interface,
but you must not call its methods yourself — FarmersDelight is the only caller. Treat the implementation as a
callback surface.

### `String name()`

The lowercase target keyword. It becomes the word an admin types:

```
/fd debugtools place keg 64 2 3
```

It is also used for tab-completion and as the extension name in `/fd stats addon <name>`. `DebugToolRegistry`
lowercases it on registration, so return it already lowercase to avoid surprises.

### `int place(Player player, Location origin, int count, int spacing, int layers, UndoSink undo)`

Called for `/fd debugtools place <name> [count] [spacing] [layers]` when `<name>` is not one of
FarmersDelight's built-in targets. Return how many blocks actually went into the world.

The command lays out the grid itself and calls `place(...)` once per prepared cell, so every call receives:

- `count`, `spacing` and `layers` all equal to `1` for that cell;
- `origin` one block **below** the prepared cell — fill `origin` raised by one on Y;
- an undo batch already open, so every `undo.capture(...)` you make joins it.

The grid comes from the admin's `count` / `spacing` / `layers` arguments, and the number of prepared cells is
capped by `max-place-count`, so one command can never ask you for more blocks than that. The cell is empty and
editable, and a capture aimed at anything else is rejected. Fill the cell, return how many blocks actually went
into the world, and the placement pattern inside it is yours to decide.

Call `undo.capture(loc)` **before** mutating each target block. `UndoSink` is a functional interface with a
single `capture(Location)`; FarmersDelight snapshots the block state at that location into the current undo
batch, so `/fd debugtools undo` can restore it. Captures that turn out not to change state are silently
no-op'd on undo.

BrewinAndChewin's keg implementation, trimmed:

```java
@Override
public int place(Player player, Location origin, int count, int spacing, int layers, UndoSink undo) {
    if (player == null || origin == null || origin.getWorld() == null) return 0;
    BlockDefinition kegBlock = CraftEngineBlocks.byId(KEG_BLOCK);
    if (kegBlock == null) {
        player.sendMessage("§c[BAC] keg block not registered with CraftEngine.");
        return 0;
    }

    int grid = Math.max(1, (int) Math.ceil(Math.sqrt(count)));
    int total = count * Math.max(1, layers);
    int placed = 0;
    for (int i = 0; i < total; i++) {
        int layer = i / count;
        int layerIndex = i % count;
        Location loc = new Location(
                origin.getWorld(),
                origin.getBlockX() + (layerIndex % grid) * spacing,
                origin.getBlockY() + 1 + layer,
                origin.getBlockZ() + (layerIndex / grid) * spacing);
        Block target = loc.getBlock();
        if (target.getType() != Material.AIR
                && !BlockStateUtils.isReplaceable(BlockStateUtils.getBlockState(target))) {
            continue;
        }
        if (undo != null) undo.capture(loc);
        if (CraftEngineBlocks.place(loc, kegBlock.defaultState(), false)) {
            placed++;
        }
    }
    return placed;
}
```

Note the `undo != null` guard — defensive, since the command always supplies a sink today.

### `int activateAt(Player player, Location location)`

Optional; default returns 0. The command passes only a position captured by the admin's latest test-placement batch. Work is sliced at 16 entries / 2ms after checking chunk residency, region ownership, build permission and the identity left by the operation. Modify that position only and check your own use permission; do not scan a world manager.

The deprecated `activate(Player)` remains for binary compatibility and is no longer called by the command. `activate all` also stays within the latest owned batch. Addon placement and activation are currently refused on Folia until the extensions support that scheduling contract.

### `List<String> status(Player player)`

Optional; default returns an empty list. Called by the **always-resident** `/fd stats addon <name>` subcommand
(the `/fd stats` overview lists every registered addon as a clickable name that drills into it), so it runs on a
release build too. Each entry is one line, sent to the player by filling the
`{line}` placeholder of the translation key `command.stats_addon_line` (`{name}` is the extension name); the
line carries whatever prefix it has itself.

```java
@Override
public List<String> status(Player player) {
    return List.of("tracked example_blocks (debug-placed): " + placed.size());
}
```

Because each line is slotted into the `command.stats_addon_line` template, avoid a literal `{line}` or `{name}`
in your text — they are treated as placeholders. Put any prefix/colour inside your own line or in that translation
key.

Returning `null` is tolerated (treated as "no lines"), but an empty list is the documented way to opt out.

### `void cleanupBeforeUndo(Location location)`

Optional; default no-op. Called during `/fd debugtools undo` for **every restored location**, before the block
data is reverted. Use it to release in-memory tracking and remove block-entity NBT belonging to your
extension, so an undone placement does not leak ghost state.

Two properties of the call site are worth knowing:

- It is invoked on **every registered extension**, not just the one that placed the block. Your
  implementation must no-op cheaply when the location is not one of yours.
- RuntimeException / LinkageError is logged and stops the undo batch, leaving unfinished entries for retry. The command does not delete a block after addon cleanup fails.

```java
@Override
public void cleanupBeforeUndo(Location location) {
    if (location == null) return;
    Iterator<Location> it = placed.iterator();
    while (it.hasNext()) {
        Location loc = it.next();
        if (loc.getBlockX() == location.getBlockX()
                && loc.getBlockY() == location.getBlockY()
                && loc.getBlockZ() == location.getBlockZ()
                && loc.getWorld() != null && loc.getWorld().equals(location.getWorld())) {
            it.remove();
            return;
        }
    }
}
```

## Threading and undo scope

Built-in work runs in slices on the player's scheduler and touches only loaded positions owned by that thread. Distant or protected positions are skipped. Paper extension callbacks must finish synchronously; deferred world edits are outside the captured count, identity checks and undo batch.

History holds at most eight batches per player. Undo checks the resulting ID / block data and container-controller identity, preserving later replacements. It cleans up test scenery and cannot recover food already collected or undo external event effects. A programmatic replacement with identical data and no controller cannot be distinguished by state alone. Stop cancels future batch work, simulation and this test's profile; already-running production continues.

## Related pages

* [Version compatibility helpers](compat-utilities.md)
* [Getting started](getting-started.md)
* [FarmersDelightApi entry point](farmersdelight-api.md)
