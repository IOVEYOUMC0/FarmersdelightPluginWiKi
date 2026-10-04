
[简体中文](../zh-cn/legacy-id-migration.md)
# Legacy id migration

Package: `com.huidu.farmersdelight.api.migration`
Class: `LegacyIdMigration` — `final`, private constructor, `@ApiStatus.NonExtendable`.

`LegacyIdMigration` lets a plugin declare item ids that used to exist and now have a replacement, so stacks
that are already in the world are rewritten the next time FarmersDelight touches them. It is a **static
facade** — there is no `getInstance()`.

## Why this exists

CraftEngine has no alias or rename mechanism for registered ids. Rename an id and the old one simply stops
existing: every stack that still carries it becomes an unknown item, in player inventories, chests, shulker
boxes, ender chests and item frames alike.

The upstream Farmer's Delight mod handles the same problem with NeoForge's `DeferredRegister.addAlias` — that
is how its 1.4 release renamed `barbecue_stick` to `cooked_meat_skewer` without breaking existing worlds.
CraftEngine has no equivalent, so FarmersDelight exposes one: declare `old id → replacement`, and the plugin
rewrites the stacks it can reach. Addons declare their own ids the same way.

Two entries from that upstream alias table show the shape of the problem:

* `barbecue_stick → cooked_meat_skewer` — an item rename;
* `basket → bamboo_basket` — upstream aliased the block **and** the item, and our pack still ships
  `farmersdelight:basket`, so this is exactly the case this layer exists for. Note that this api migrates
  **items**: block-level ids are not covered (see *Limits*).

## The api

```java
public static void registerItem(Key legacyId, ItemStack replacement);
public static void registerItem(Key legacyId, Key currentId);
public static boolean isLegacy(ItemStack stack);
public static ItemStack migrate(ItemStack stack);

// read-only helpers
public static String resolveId(String legacyId);   // the id it ends up as, or null
public static boolean isEmpty();                   // nothing registered yet
public static int size();                          // registered legacy ids
public static int conflictCount();                 // registrations rejected as duplicates
```

`Key` is CraftEngine's `net.momirealms.craftengine.core.util.Key` — the same type `ContentRegistration` uses.

### registerItem(Key legacyId, ItemStack replacement)

The common form. Declares that stacks carrying `legacyId` become `replacement`, whose meta is also the
**template** for the migrated stack: if you need a custom name, lore, enchantments or damage on the result,
build the stack you want and pass it here.

Ignored (a silent no-op) when `legacyId` is `null`, when `replacement` is `null` or air, or when the
replacement's CraftEngine id cannot be resolved. The template is stored under the id the replacement resolves
to, so chained mappings share one template.

### registerItem(Key legacyId, Key currentId)

The convenience overload: the migrated stack is created from `currentId`'s own definition, so it carries no
custom meta. The amount always comes from the old stack.

### isLegacy(ItemStack stack)

True when the stack carries an id that has been registered as legacy. False for `null`, for an empty stack, and
for every stack while nothing is registered.

### migrate(ItemStack stack)

Returns the replacement for a legacy stack, or the stack itself. The contract:

* `null` comes back as `null`; an empty stack comes back unchanged.
* An id that is not registered — including an already-migrated stack — comes back as **the same instance**.
  Use identity (`migrated == original`) as the change signal; that is what the plugin's own hooks do.
* A registered id becomes the replacement item with the **amount kept**.
* **Persistent data is merged**: entries the replacement already carries win — this includes CraftEngine's own
  id key, so the result really is the new item — and entries only the old stack had are carried over.
* **Not copied:** custom name, lore, enchantments, damage. Pass a fully prepared replacement stack to
  `registerItem(Key, ItemStack)` if you need those.
* **Idempotent:** the new id is not registered as legacy, so calling it again changes nothing.
* If the replacement has no usable definition, the old stack is returned unchanged rather than deleted.
* Chains resolve in one call — `A → B → C` becomes `C` — bounded to 8 hops, so a cycle terminates.
* With an empty table it returns immediately, which is why the automatic hooks cost nothing until an addon
  registers something.
* **No world access.** It only reads the stack it is given, so any thread or region may call it. The automatic
  hooks always call it on the owner of the inventory, block or entity.

## The precondition: keep the old definition in the pack

**The legacy id must still have an item definition in the pack that produced it.** CraftEngine resolves an id
against its loaded definitions; once the old definition is gone, an existing stack is turned into an *unknown
item* before this layer can see a usable id, and `isLegacy` will never match. A migration table whose old id
has no definition is dead code.

So the correct order for a rename is:

1. **Keep the old definition in the pack** — even if it now points at the same model and texture as the new
   item. Ship it in the same release as the new id.
2. Register the mapping in `onEnable` (or wherever you register your content), again in the same release.
3. Only then stop *handing the old item out*: remove the recipes/tags/lang entries that produce or name it, so
   no new old-id stacks are created. The definition itself may stay indefinitely — an unused definition is
   cheap, and removing it early is what breaks migration.

## Automatic hooks

Five hooks rewrite stacks as they come into view. You do not schedule anything.

| Moment | What is rewritten | Runs on |
| --- | --- | --- |
| Player join | the player's inventory and ender chest | the player's thread |
| Container open | the opened inventory, once per open (not once per click) | the region owning the block |
| Our own container GUI | the mirror in the GUI's top inventory, at its load boundary | the viewing player's thread |
| Item spawn | the stack that just dropped | the item entity's thread |
| Startup sweep | block-entity inventories in already-loaded chunks | one dispatch per loaded chunk |

Every hook decides through the same rule: the work runs **inline** when the current thread already owns the
player, item or block, and otherwise it is handed to the owning region as **exactly one** dispatch. A hook
never reads a container it does not own, and none of them use `Bukkit.getScheduler`. On one thread (a plain
Paper server) this is all inline work; on Folia the cross-region case costs one scheduled task. The startup
sweep walks loaded chunks once and re-checks each chunk inside its own region, so an unloaded chunk is skipped.

### Which containers are touched

A container is rewritten only when its **owner can be named**:

* a `BlockState` holder — the block's location;
* either half of a `DoubleChest` — the left side is preferred, and only for determinism;
* one of FarmersDelight's own container GUIs — the viewing player.

Any other holder — a third-party holder plugin, a merchant, a custom `InventoryHolder` this plugin cannot
attribute — is **skipped**, and the open is not recorded as migrated, so the next open tries again. Skipping
is deliberate: reading a container from whichever thread happens to see the event, without knowing which
region owns it, is the one thing these hooks must not do. A skipped container therefore migrates late (on a
later open), or not at all if it is never opened again.

### What the hooks do not reach

The hooks cover only the storage FarmersDelight itself touches: player inventory and ender chest, an opened
container, this plugin's container GUI boundary, stacks that spawn in the world, and block-entity inventories
in already-loaded chunks at startup. **Your addon's own storage is not on that list** — a custom GUI cache, a
database column, a recipe-book cache, or item references in your own YAML.

Those need an explicit call at your own boundary:

```java
// A stack in your own storage: migrate() returns the same instance when nothing matched.
ItemStack fixed = LegacyIdMigration.migrate(storedStack);
if (fixed != storedStack) {
    saveToMyStorage(fixed);
}

// An id string (a config value or a database column holding "myaddon:old_widget").
String current = LegacyIdMigration.resolveId(storedId);   // null when it is not a legacy id
if (current != null) {
    saveToMyStorage(current);
}
```

## Config switches

`config.yml`:

| Switch | Effect |
| --- | --- |
| `legacy-id-migration.enabled` | `false` stops the five automatic hooks |
| `legacy-id-migration.log-summary` | `true` logs one aggregated line per run |

`log-summary` writes `Legacy id migration: rewrote N stack(s) across M inventory(ies).`, and only when the
total grew since the previous line, so a busy server does not get a line per inventory.

**Switching `enabled` off does not disable the api.** `LegacyIdMigration.migrate(stack)` stays available
either way — that is why an addon should migrate at its own storage boundaries instead of relying on the
hooks alone. The hooks are a convenience for the storage FarmersDelight already touches.

**Switching it back on does not replay the startup sweep.** `enabled` is re-read by `/fd reload`, but that
path only re-reads the two values; the sweep runs at startup and is not run again. After a `false → true`
change, stacks sitting in already-loaded chunks therefore wait for their next natural contact — a player
joining, or someone opening the container. To force a full pass on a live server, restart: startup is the
only automatic sweep.

## Example

```java
import com.huidu.farmersdelight.api.migration.LegacyIdMigration;
import net.momirealms.craftengine.core.util.Key;

// inside onEnable(), i.e. inside a method body
LegacyIdMigration.registerItem(
        Key.of("myaddon:old_widget"),   // the id that may exist in old worlds
        Key.of("myaddon:new_widget"));  // what it becomes

// When the replacement needs its own name, lore or enchantments, pass a prepared stack:
LegacyIdMigration.registerItem(Key.of("myaddon:old_gem"), preparedNewGem());
```

At a boundary only your addon knows about:

```java
ItemStack stored = readFromMyCache();
ItemStack fixed = LegacyIdMigration.migrate(stored);
if (fixed != stored) {
    writeBackToMyCache(fixed);   // identity is the change signal
}
```

Call these from **inside a method body** — `onEnable`, a listener, a command handler. Do not reference
`LegacyIdMigration` from a field initializer or a `static final` constant: that loads the class during your
addon's own class initialization, where a failure (`NoClassDefFoundError`) aborts the initializer and silently
leaves whatever you were wiring disabled. The same rule applies to every api class — see
[Version compatibility helpers](compat-utilities.md).

## Ids, chains and conflicts

Ids are **case-sensitive** and only trimmed (MMOItems ids carry an upper-case type segment). `null` and blank
ids are ignored.

Registering the same legacy id twice with a different target keeps the **first** mapping and reports the second
once, through the plugin's logger — two addons cannot silently fight over one id. Registering the same mapping
twice is a no-op. `conflictCount()` tells you how many registrations were rejected that way.

## Limits

* **Items only.** Renames of *block* ids are not covered today.
* **Not a persistent alias.** It rewrites a stack when FarmersDelight (or your own call) touches it; a stack
  that is never loaded is never rewritten. It is not a lookup that makes the old id resolve forever.
* **No data-pack work.** Recipes, tags, advancements and language keys that name the old id are files, not
  stacks — update them separately.
* **It cannot resurrect a deleted definition.** Remove the old item definition and the old stacks are unknown
  before migration can help, which is exactly why the precondition above matters.
* **Your own storage is not covered.** If a stack or an id lives somewhere only your addon can read — a cache,
  a database, a custom GUI, your own YAML — call `migrate(ItemStack)` or `resolveId(String)` there yourself;
  the hooks cannot see it.
* **Re-enabling the switch does not re-sweep.** See *Config switches*: startup is the only automatic pass over
  already-loaded chunks, so a `false → true` change leaves those stacks unmigrated until they are touched.
* **The amount and the persistent-data merge are only verifiable in-game.** The offline tests lock the mapping
  rules and the sequence of calls a migration makes, but they cannot land or read the result back: an
  `ItemStack` subclass can be constructed without a server, yet `setAmount`, `getType`, `getItemMeta` and the
  persistent-data container are Craft-backed and throw without one. Treat "the amount is kept" and "the data
  merged" as behaviour to confirm on a test server.

## Availability

No `hasFeature` id covers this class, so there is no probe for it. It is part of the published `api.**` surface
of the builds that ship it, and an older build simply lacks the class — any path that references it then fails
with `NoClassDefFoundError`. Either require a build that has it, or keep the reference inside a method body you
only reach after checking that the class is there.

## Related pages

* [Content registration](content-registration.md)
* [Items](items.md)
* [Version compatibility helpers](compat-utilities.md)
