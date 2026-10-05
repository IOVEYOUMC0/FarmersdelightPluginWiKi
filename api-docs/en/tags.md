
[简体中文](../zh-cn/tags.md)
# Tags and knife-mineable blocks

Package: `com.huidu.farmersdelight.api.tag`

Two classes: `FarmersDelightTags` — the tag ids this plugin uses, as plain `String` constants — and
`KnifeMineableBlocks`, a read-only query over a CraftEngine block definition's declared tags.

Types covered here:

| Type | Kind | `@ApiStatus` |
| --- | --- | --- |
| `KnifeMineableBlocks` | static facade, private constructor | `@NonExtendable` |
| `FarmersDelightTags` | static constant holder, private constructor | — |

## Querying "mineable with a knife"

```java
public static boolean isKnifeMineable(Key blockId);
```

True when the CraftEngine block definition for `blockId` declares either `farmersdelight:mineable/knife`
(`FarmersDelightTags.BLOCK_MINEABLE_WITH_KNIFE`) or the common `c:mineable/knife`
(`FarmersDelightTags.COMMON_MINEABLE_WITH_KNIFE`). Both names count, so a pack written for this plugin's family
and one using the common namespace give the same answer.

### Why a CraftEngine `Key` and not a `Block`

The parameter is the block's **CraftEngine id**, not a Bukkit `Block`. Resolving a `Block` to its CraftEngine id
means reading the world's custom block state — a region-bound read — and a Bukkit object cannot be handed to
another region. Taking the id keeps this a **pure definition lookup**: it reads CraftEngine's in-memory
registries and never touches a world, chunk, block entity or block state, so **any thread and any region may
call it**.

### Contract

* A `null` id is `false`.
* An id with no CraftEngine block definition is `false`.
* Any lookup failure is `false`: a `RuntimeException` or a `LinkageError` (CraftEngine not up yet, a definition
  caught mid-reload, a CraftEngine class this server cannot load) is swallowed. **Nothing is thrown to the
  caller** — "not declared" is the safe answer.
* A definition that declares no tags, or declares only other tags, is `false`.
* No world access, no scheduling, and no state kept between calls.

### What it does not do

**It only reports data.** Three consequences matter:

* **It does not change mining speed.** Break speed comes from vanilla's `minecraft:mineable/*` tags plus the
  held item's digger rules, and CraftEngine's own tool check uses the definition's
  `requireCorrectTool()` / `requiredBreakPower()` / `isCorrectTool(..)` settings — none of those read the
  declared `tags`. Use `require_correct_tool` in the pack when a tool requirement has to gate something.
* **It is not wired into any behaviour today.** Nothing calls it while a block is being broken. The drop path is
  still the pack's `match_item` rules.
* **Wiring a drop to it in Java would double the drops** on every block whose pack already has such a rule. So
  before connecting anything, you need a way to know whether a block already has a CraftEngine drop rule — this
  api has no such predicate today, so do not connect it.

Treat it as a read-only fact query for addons and for this plugin's own decisions.

## Example

```java
import com.huidu.farmersdelight.api.tag.KnifeMineableBlocks;
import net.momirealms.craftengine.core.util.Key;

// You already hold the CraftEngine block id — from your own pack or config.
if (KnifeMineableBlocks.isKnifeMineable(Key.of("myaddon:crate"))) {
    // The pack declared this block mineable with a knife. The drop behaviour is still the pack's
    // `match_item` rules, so do not add a Java drop on top of them.
}
```

Pass an id you already have. Do not build the `Key` from `FarmersDelightBlocks.blockIdOf(block)` while on
another region: that resolution is exactly the region-bound world read this api is shaped to avoid.

## Tag constants

`FarmersDelightTags` holds the tag ids this plugin uses, so addons can name them without retyping strings. They
are plain `String` constants: `farmersdelight:...` for the plugin's own tags and `c:...` for the common tags it
also honours.

```java
// block tags
public static final String BLOCK_HEAT_SOURCES         = "farmersdelight:heat_sources";
public static final String BLOCK_HEAT_CONDUCTORS      = "farmersdelight:heat_conductors";
public static final String BLOCK_MINEABLE_WITH_KNIFE  = "farmersdelight:mineable/knife";
public static final String BLOCK_MUSHROOM_COLONIES    = "farmersdelight:mushroom_colonies";
public static final String BLOCK_CABINETS             = "farmersdelight:cabinets";
public static final String BLOCK_COMPOST_ACTIVATORS   = "farmersdelight:compost_activators";
// ... plus drops_cake_slice, tray_heat_sources, planted_from_below, feasts, pies, straw_blocks,
//     terrain, unaffected_by_rich_soil, cabinets/wooden, ropes, wild_crops, campfire_signal_smoke

// item tags
public static final String ITEM_KNIVES                = "farmersdelight:tools/knives";
public static final String ITEM_KNIFE_ENCHANTABLE     = "farmersdelight:enchantable/knife";
public static final String ITEM_MILK                  = "farmersdelight:milk";
public static final String ITEM_FLAT_ON_CUTTING_BOARD = "farmersdelight:flat_on_cutting_board";
// ... plus snacks, meals, drinks, sweets, feasts, pies, serving_containers, straw_harvesters,
//     cabinets, cabinets/wooden, canvas_signs, hanging_canvas_signs, mushroom_colonies, wild_crops

// entity type tags
public static final String ENTITY_HORSE_FEED_TEMPTED   = "farmersdelight:horse_feed_tempted";
// ... plus dog_food_users, horse_feed_users

// common (c:) tags
public static final String COMMON_MINEABLE_WITH_KNIFE  = "c:mineable/knife";
public static final String COMMON_TOOLS_KNIFE          = "c:tools/knife";
public static final String COMMON_CROPS_CABBAGE        = "c:crops/cabbage";
// ... plus crops/tomato, crops/onion, crops/rice, storage_blocks/cabbage, storage_blocks/rice
```

The class itself is the full list, grouped by prefix: `BLOCK_*`, `ITEM_*`, `ENTITY_*`, `COMMON_*`. A constant is
only the id string — using it changes nothing about how the tag resolves.

## Availability

No `hasFeature` id covers this class, so there is no probe for it. It is part of the published `api.**` surface
of the builds that ship it; an older build simply lacks the class, and any path that references it then fails
with `NoClassDefFoundError`. Either require a build that has it, or keep the reference inside a method body you
only reach after checking. As with every api class, call it from **inside a method body** (`onEnable`, a
listener, a command handler), never from a field initializer or a `static final` constant.

`farmersdelight:mineable/knife` is read from the block definition's **default state**. Under CraftEngine
**26.9.1** — the version this build's default dependency is pinned to — that is
`BlockDefinition.defaultState().settings().tags()`. The api surface does not change with that path.

## Related pages

* [Blocks and stations](blocks-and-stations.md)
* [Items](items.md)
* [Content registration](content-registration.md)
