
[简体中文](../zh-cn/compat-utilities.md)
# Version compatibility helpers

`com.huidu.farmersdelight.api.util` contains the small helpers that exist so an addon can support the whole
supported Minecraft range from a single compiled jar. `CompatAttributes` and `CompatItemMeta` paper over
Bukkit API changes; `TooltipUtils` and `CeItemInterop` deal with CraftEngine's own item wrapper.

The supported floor is 1.21.5, so both compatibility shims now resolve on every supported server. They are
kept because they are published API and because they still absorb the attribute-registry rename and any
future removal of `setItemModel`.

`DebugToolExtension` and `DebugToolRegistry` also live in this package but are covered in
[Debug tools](debug-tools.md). `PluginManagerGuard` is covered in [Getting started](getting-started.md).

These are the single shared implementations. Do not keep your own copy — BAC originally did, and its local
`CompatAttributes` / `CompatItemMeta` are now thin delegating shims kept only so its existing imports still
resolve.

## CompatAttributes

`@ApiStatus.Internal`, `@ApiStatus.NonExtendable`, final, two public constants.

```java
public static final Attribute MAX_HEALTH;
public static final Attribute ATTACK_SPEED;
```

Minecraft renamed the attribute registry entries: older servers expose the enum constant `GENERIC_MAX_HEALTH`
under the registry key `generic.max_health`, newer ones use `MAX_HEALTH` and `max_health`. Referencing either
enum constant directly in your code makes the class fail to load on the other side of the rename.

Resolution goes through `Registry.ATTRIBUTE` by key first — modern spelling, then legacy — and falls back to a
reflective field lookup. No version-specific enum constant is ever referenced directly, so the class loads on
every supported version.

**A constant is null when the running server exposes neither spelling.** You must null-check before passing it
to `getAttribute`. This is BAC's real usage:

```java
AttributeInstance maxHealthAttr = player.getAttribute(CompatAttributes.MAX_HEALTH);
if (maxHealthAttr == null) {
    return;
}
```

**`@ApiStatus.Internal` is not decoration here — reference these constants only from inside a method body.**
They are static fields, so naming one from your own field initializer, static block or `static final` constant
makes this class load during *your* addon's class initialization. An addon runs under its own class loader, and
a failure there (`NoClassDefFoundError`) aborts that initializer and silently leaves whatever you were wiring
up disabled — the plugin loads, the feature just never works. A method body keeps the load on a path your addon
can handle; for a lookup this small, a self-contained copy in the addon is the safer answer still.

## CompatItemMeta

`@ApiStatus.NonExtendable`, final, two static methods.

| Method | Behaviour |
| --- | --- |
| `isSupported()` | `true` when the running server has `ItemMeta.setItemModel`. |
| `setItemModel(ItemMeta meta, NamespacedKey key)` | Applies the `item_model` component, or does nothing. |

`ItemMeta.setItemModel` exists from Minecraft 1.21.4 on, which is below the supported floor, so the
reflective lookup (resolved once into a static field) succeeds on every supported server. The shim is kept so
an addon compiled against an older api jar keeps working, and so the call degrades instead of throwing if a
fork removes the method.

`setItemModel` is a **silent no-op** in three cases: the method is unavailable, `meta` is null, or `key` is
null. It also swallows `ReflectiveOperationException`. It never throws and never reports failure, so if the
model must be visible you cannot detect the miss from the return value — there isn't one.

BAC's real call site guards only the key parsing and lets the no-op absorb the version difference, setting
custom model data independently so older servers still have something to render:

```java
if (customModelData != null) {
    meta.setCustomModelData(customModelData);
}
if (itemModel != null && !itemModel.isBlank()) {
    NamespacedKey modelKey = NamespacedKey.fromString(itemModel.toLowerCase(Locale.ROOT));
    if (modelKey != null) {
        CompatItemMeta.setItemModel(meta, modelKey);
    }
}
```

`isSupported()` is there for when you need to branch to a genuinely different fallback instead. No current
consumer calls it — every real call site relies on the silent no-op.

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

## Related pages

* [Getting started](getting-started.md)
* [Content registration](content-registration.md)
* [Debug tools](debug-tools.md)
* [Items](items.md)
