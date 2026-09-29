
[简体中文](../zh-cn/knife-drops.md)
# Knife Drops

Package: `com.huidu.farmersdelight.api.loot`
Class: `FarmersDelightKnifeDrops` — `final`, private constructor, `@ApiStatus.NonExtendable`.

Registration of extra mob drops harvested with a knife: the rule set behind FarmersDelight's ham,
leather, feather and string drops. The rules that ship with the plugin are data, not Java — they live in
the bundled CraftEngine pack file `vanilla_loots.yml` (`farmersdelight:ham_from_pig` and the like), so a
server owner tunes them there. This facade is for plugins that need to add rules at runtime — an addon
shipping its own butchering items, a quest plugin adding a conditional drop.

## How a rule fires

A rule fires when a player kills an **adult** entity of the matching type while holding a matching tool,
and a chance roll passes.

The rolled item is **contributed to the death event's drop list** rather than spawned directly, so other
plugins' loot handling sees it like any vanilla drop. The pack-defined rules do the same, through the
death event as well.

## API

```java
public static boolean register(String entityType, String normalItemId, String burningItemId,
                               double chance, double lootingMultiplier,
                               List<String> toolItems, List<String> toolTags);

public static boolean register(String entityType, String normalItemId, String burningItemId,
                               double chance, double lootingMultiplier);

public static boolean     unregister(String entityType);
public static Set<String> entityTypesWithRules();
public static boolean     hasRule(String entityType);
```

### register

| Parameter | Meaning |
| --- | --- |
| `entityType` | Bukkit `EntityType` name, case-insensitive — `"pig"`, `"COW"` both work. Trimmed and lowercased internally |
| `normalItemId` | item id to drop, `"ns:id"`. `"minecraft:air"` or `null` disables the drop |
| `burningItemId` | item id to drop instead when the entity dies on fire, or `null` to always use `normalItemId` |
| `chance` | base drop probability, 0..1 |
| `lootingMultiplier` | added to `chance` per Looting level on the killing tool; 0 to ignore Looting |
| `toolItems` | item ids that count as the harvesting tool. Empty or `null` falls back to the plugin's knife definition |
| `toolTags` | item tag ids that count as the harvesting tool, written **without** the leading `#`. Empty or `null` falls back to the plugin's knife definition |

Returns `false` when FarmersDelight is unavailable (not loaded, or not enabled) or `entityType` is null
or blank. Otherwise `true`.

**There is one rule per entity type, not a list.** Registering for a type that already has a rule
replaces it.

The five-argument overload is the seven-argument one with `null, null` for the tool matchers, so the
rule uses the plugin's knife definition: the configured knife items and tags plus any CraftEngine tag an
item declares itself — the same check the pack rules use.

```java
import com.huidu.farmersdelight.api.loot.FarmersDelightKnifeDrops;

// Uses whatever the server has configured as "a knife".
FarmersDelightKnifeDrops.register("rabbit", "myaddon:rabbit_cutlet", null, 0.35D, 0.10D);

// Only this addon's own butchering tools trigger it, and burnt mobs drop the cooked version.
FarmersDelightKnifeDrops.register(
        "sheep",
        "myaddon:raw_mutton_strips",
        "myaddon:seared_mutton_strips",
        0.5D,
        0.05D,
        List.of("myaddon:cleaver"),
        List.of("myaddon:butchering_tools"));
```

### unregister

Removes a rule previously added by `register`. Returns `true` when a rule registered *by this facade*
existed for that entity type.

It only touches rules registered through this facade. The pack-defined drops are separate CraftEngine
loot entries and cannot be removed from here — override those by editing the pack, or by adding your own
loot source.

Call it on your plugin's disable.

### Queries

`entityTypesWithRules()` — the lowercase entity type names that currently have a rule **registered through
this facade**. Returns an empty set when FarmersDelight is unavailable.

`hasRule(entityType)` — true when such a rule exists for that type. Case-insensitive.

Both read the facade's own rule map, so they do not report the drops the pack defines (pig, cow, chicken
and the rest). Use them to check your own registrations.

## Surviving `/fd reload`

Rules registered here persist across `/fd reload`: the reload no longer rebuilds a rule map from a config
file, so the facade's registrations simply stay in place.

So a registration outlives an admin's reload without your addon having to listen for
`FarmersDelightReloadEvent`.

Because there is no config file behind the map any more, an `unregister` takes effect immediately, and a
rule you register for an entity the pack already covers (pig, cow, chicken) means that mob drops **both**
your item and the pack's.

## Threading

`register` / `unregister` write to a `ConcurrentHashMap` and are safe to call from `onEnable`, a reload
handler, or `onDisable`. The queries read the live map.

There is no documented threading contract for registering from an async thread mid-tick: the map is
concurrent, so it will not corrupt, but do not assume a death event handler in the same tick will observe
the new rule. Register at startup or on reload and this does not arise.

## Related pages

* [Items](items.md)
* [Events](events.md)
