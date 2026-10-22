
[简体中文](../zh-cn/compat-utilities.md)
# Version compatibility helpers

`com.huidu.farmersdelight.api.util` provides CE item interop, optional timing and published compatibility interfaces. Paper 1.21.5 is the minimum; new code uses Bukkit/Paper APIs already available there. `CompatAttributes` and `CompatItemMeta` retain existing signatures with direct implementations.

See [debug tools](debug-tools.md) for `DebugToolExtension` and `DebugToolRegistry`, and [getting started](getting-started.md) for `PluginManagerGuard`.

## PerformanceProbe

`PerformanceProbe.begin(Operation)` returns `null` when the operation is not sampled, without reading a clock or allocating a sample. Active samples implement `AutoCloseable`; Java try-with-resources accepts null and records exceptional exits. Each sample records once and cannot pollute a later session. Close a sample on its executing thread.

`item(ItemStack)` / `items(ItemStack[])` delegate to CE's existing codec; `holder(Inventory)` always uses Paper `getHolder(false)`. These helpers do not schedule work, change ownership or replace locks, and do not authorize asynchronous live-inventory access. Addons import only `api.util.PerformanceProbe` and must update alongside this FD build.

```java
import com.huidu.farmersdelight.api.util.PerformanceProbe;

try (PerformanceProbe.Sample ignored = PerformanceProbe.begin(PerformanceProbe.Operation.SERIALIZATION)) {
    // Keep the original serialization and error handling here.
}
```

Operations are `SERIALIZATION`, `ITEM_SERIALIZATION`, `SERIALIZATION_LOCK`, `INVENTORY_HOLDER`, `RECIPE_MATCH`, `STATION_TICK` and `GUI_REFRESH`. Begin a lock sample before entering synchronized and close it on the first line inside, so encoding is excluded. `RECIPE_MATCH` wraps one cooking pot / cutting board lookup, `STATION_TICK` one cooking pot tick pass and `GUI_REFRESH` one recipe-viewer screen refresh. `FarmersDelightApi.get().performanceTiming(operation)` returns a session-bound recorder, or null while inactive/unavailable; do not retain it long-term. See [debug tools](debug-tools.md).

## CompatAttributes

The internal `MAX_HEALTH` and `ATTACK_SPEED` aliases now refer directly to Bukkit constants. New addons use `Attribute.MAX_HEALTH` and `Attribute.ATTACK_SPEED` without registry probing or a separate shim. `getAttribute` still returns null when an entity lacks that attribute.

```java
import org.bukkit.attribute.Attribute;
import org.bukkit.attribute.AttributeInstance;

AttributeInstance maxHealth = player.getAttribute(Attribute.MAX_HEALTH);
if (maxHealth == null) return;
```

## CompatItemMeta

| Method | Behaviour |
| --- | --- |
| `isSupported()` | Always true within the supported range; retained for existing callers. |
| `setItemModel(ItemMeta meta, NamespacedKey key)` | No operation for null meta/key; otherwise calls the Paper API directly. |

New code uses `meta.setItemModel(modelKey)` directly. The compatibility method no longer swallows invocation errors. Use `CustomModelDataComponent` for model numbers rather than the deprecated integer setter.

```java
import org.bukkit.inventory.meta.components.CustomModelDataComponent;
import java.util.List;

CustomModelDataComponent data = meta.getCustomModelDataComponent();
data.setFloats(List.of(customModelData.floatValue()));
meta.setCustomModelDataComponent(data);
meta.setItemModel(modelKey);
```

## CeItemInterop

`@ApiStatus.NonExtendable`, final, three static conversions between CraftEngine's
`net.momirealms.craftengine.core.item.Item` wrapper and Bukkit's `ItemStack`. These are the conversions the
plugin's container and recipe code performs on every stored item, so each one lives here instead of being
copied per call site.

| Method | Behaviour |
| --- | --- |
| `asBukkitStack(Item item)` | The Bukkit stack, or `null` when `item` is `null` **or empty**; the count is not inspected |
| `toBukkitStack(Item item)` | The Bukkit stack for an already-checked item — **no empty guard** |
| `normalize(Item item)` | A defensive copy of the item, or `Item.empty()` |

`asBukkitStack` is the guarded one: a null or empty item returns `null`, and everything else goes through
CraftEngine's own `ItemStackUtils.getBukkitStack`. Nothing here resolves CraftEngine's readiness, so a caller
that can run before CraftEngine is ready keeps its own readiness guard, and a failure inside the conversion
propagates to the caller rather than being swallowed.

`toBukkitStack` is the same conversion **without** the empty-item guard, for a caller that inspects the
converted stack itself or substitutes its own fallback. A null item still returns `null`; every other item is
handed to CraftEngine unchanged — including an empty one, which converts to an empty `ItemStack` rather than
to `null`. Pick the one that matches what your call site does with nothing.

`normalize` returns something you can safely keep: `Item.empty()` when the argument is null, empty, or has a
count that is not positive, and otherwise a **copy** with the same count (`copyWithCount`) rather than the
instance you passed. That copy is why it is safe to store the result while the caller keeps mutating its own
item.

```java
Item stored = CeItemInterop.normalize(incoming);   // never null; a copy when there is content
ItemStack bukkit = CeItemInterop.asBukkitStack(stored);   // null for the empty case
```

## TooltipUtils

Final, no `@ApiStatus` annotation, one static method:

```java
public static void hideDurabilityLine(Item wrapped)
```

Note the parameter type: this is CraftEngine's `net.momirealms.craftengine.core.item.Item` wrapper, **not**
Bukkit's `ItemStack`. Callers are already in CraftEngine territory when they need this, so the signature does
not pretend otherwise. Wrap a stack with `BukkitItemManager.instance().wrap(stack)` first.

The method adds `minecraft:damage` and `minecraft:max_damage` to
`minecraft:tooltip_display.hidden_components`, merging with any pre-existing hidden entries rather than
overwriting them. Use it when an item's damage value encodes something other than tool wear — a fill bar, a
serving count — so the player is not shown a meaningless "Durability: X / Y" line.

Mutation is applied to the wrapper, which writes through to the stack you wrapped. Clone first if the caller's
stack must stay untouched. FDAddonTemplate shows the whole pattern, encoding a fill level into the damage bar:

```java
ItemStack copy = stack.clone();
Item wrapped = BukkitItemManager.instance().wrap(copy);
wrapped.maxDamage(capacity);
wrapped.damage(Math.max(1, capacity - Math.min(capacity, amount)));
TooltipUtils.hideDurabilityLine(wrapped);
return copy;
```

The `Math.max(1, ...)` is load-bearing: vanilla hides the damage bar entirely at damage 0, so a completely
full meter would show no bar at all without the clamp.

## Startup pack utilities

`DatapackSupport` exposes `managedPack`, `startupConfiguration`, `discoverPack`, `migrateLegacyPack` and `archiveLegacyPack`. Call `discoverPack` only from Paper's bootstrap datapack discovery event; the official `DatapackRegistrar` loads newly discovered managed packs and respects saved manual disablement. Configuration parsing fails explicitly, file writes use temporary-file replacement, and legacy world packs are copied into the managed directory before the old directory is removed, without automatic backups. `archiveLegacyPack` remains an explicit utility and is not called by automatic migration. Existing backups are left untouched.

`DamageTypePack.prepare(configuration, defaults, pack, description, version)` generates registry JSON and metadata from damage YAML. `saveConfiguration(configuration, pack)` imports existing JSON only when the YAML is absent. The configuration has `damage-types` and `tags` mappings, keyed by full registry IDs and using Minecraft's field names. Edit YAML and fully restart; JSON is generated output. FD, BAC and CAB share this utility; both addons declare FD as a bootstrap dependency. New addons should register initial definitions in their own bootstrap; dynamic onEnable registrations still require the next startup.

This uses Paper datapack discovery and still loads JSON. Direct registry registration uses `freeze()` in 1.21.5 and `compose()` from 1.21.6 onward, so a static call to the new method cannot cover the whole supported range.

## Related pages

* [Getting started](getting-started.md)
* [Content registration](content-registration.md)
* [Debug tools](debug-tools.md)
* [Items](items.md)
