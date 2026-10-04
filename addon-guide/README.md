---
icon: puzzle-piece
---

# Addon Guide

[简体中文](README.zh-cn.md)

FarmersDelight is the core plugin of a small family of CraftEngine content packs. Each addon is an
independent plugin that **depends on FarmersDelight + CraftEngine** and layers a themed set of blocks,
items, recipes and mechanics on top of them. This page explains, for a server admin, **what each addon
adds, how it hooks into FarmersDelight, and which config knobs it exposes** — so you can understand and
tune the behaviours the addons register.

Most addons are faithful ports of a Forge/Fabric mod; Villagers' Delight is original to this project. None
of them replace FarmersDelight — they extend it. Attribution and licensing are per addon, exactly as the
upstream projects state them:

| Addon | Upstream mod | Authors | Upstream license |
| ----- | ------------ | ------- | ---------------- |
| FarmersDelight (this core plugin) | Farmer's Delight | vectorwing | MIT |
| Brewin' And Chewin' (BAC) | Brewin' And Chewin' | ProbablyEyes (owner), Umpaz, MerchantCalico, RaymondBlaze, Farcr | MIT, `Copyright (c) 2022 Umpaz` |
| Barbeque's Delight (BBQD) | Barbeque's Delight | MaoMao, lcy0x1 | MIT (`LICENSE`, `Copyright (c) 2024 MaoMao`); the mod's `mods.toml` declares LGPL-2.1 instead — see the notes |
| Crabber's Delight (CD) | Crabber's Delight | AlabasterLeking | MIT, declared by the upstream build metadata only |
| End's Delight (ED) | End's Delight | FoggyHillside | MIT, `Copyright (c) 2022 FoggyHillside` |
| Villagers' Delight (VD) | — (original to this project) | this project | not applicable (no upstream mod) |

Notes on the ones that need them:

* **BAC** — our CraftEngine content is based on the revision whose `LICENSE` read "MIT License, Copyright (c)
  2022 Umpaz", and that licence travels with this port (it ships in the addon jar's `NOTICE.txt`). Upstream
  **deleted its `LICENSE` file on 2026-08-24** (commit `ed58394`), so later upstream versions no longer carry
  it; this port stays under the MIT revision it was built from. The author list above is the union of the two
  sources, written with canonical spellings: `MerchantCalico` is the same person as the GitHub handle
  `MerchantPug`, and `Probleyes` — the spelling the jar's `NOTICE.txt` uses — is an older spelling of
  `ProbablyEyes`.
* **BBQD** — the upstream repository's `LICENSE` is MIT (`Copyright (c) 2024 MaoMao`) and Modrinth also lists
  the mod as MIT, while its `META-INF/mods.toml` declares `license="LGPL-2.1"`. This project distributes its
  port under the repository `LICENSE` (MIT). The two upstream statements contradict each other; that is an
  upstream inconsistency, not a choice made here.
* **CD** — upstream ships **no `LICENSE` file and no copyright line**; MIT is declared only by its build
  metadata (`gradle.properties: mod_license=MIT License`, `mod_authors=AlabasterLeking` and the matching
  `neoforge.mods.toml`), so treat it as a metadata declaration rather than a signed licence text.

***

## How an addon hooks into FarmersDelight

All the CraftEngine-based addons (everything except VillagersDelight) share the same registration model.
Understanding it explains **when** their content appears and **why a reload sometimes needs a specific
command**.

### Lifecycle — when content registers

CraftEngine loads its items in a **deferred pass after every plugin has enabled**, so an addon cannot
resolve custom item icons/results during its own `onEnable`. Registration is therefore split across
several lifecycle points:

| Moment | What an addon does here |
| ------ | ----------------------- |
| **`onEnable`** | Load `config.yml` + language files, register its CraftEngine block behaviours and item functions (so the pack parses), register its **special-recipe cards**, mob-drop / knife-drop rules, and its Bukkit listeners. All of these resolve items lazily, so they are safe before CE finishes loading. |
| **`FarmersDelightWarmupEvent`** | Fires once CraftEngine items are loaded (first boot, and again after every `/ce reload`). Addons register their **advancements** (icons resolve now) and their **cooking-pot / cutting-board recipes** into FarmersDelight here. |
| **`CraftEngineReloadEvent`** | Re-registers recipes after a CraftEngine reload. |
| **`FarmersDelightReloadEvent`** (`/fd reload all`) | Re-syncs the addon's config-driven state. |

An addon that would otherwise register the same content both eagerly (at `onEnable`) and again on warmup
gates the eager path on `FarmersDelightApi.get().isContentLoaded()` — true only once CraftEngine items are
loaded — so nothing registers (or logs) twice on a normal start.

**Practical upshot:** after editing an addon's **recipes or CraftEngine resources**, run `/ce reload`
(re-fires warmup). After editing an addon's **`config.yml`**, run `/fd reload all` (re-fires the addon
reload). A full restart always does both.

### Where an addon's behaviour actually lives

| Mechanism | Used for | Where you configure it |
| --------- | -------- | ---------------------- |
| CraftEngine item `events: - on: consume` + functions | On-eat effects (booze/tipsy, teleport foods, nourishment) | The item definition in `plugins/CraftEngine/resources/<namespace>/…/items.yml` |
| CraftEngine block `behavior:` | Custom blocks (keg, grill, note board, stove-likes) | The block definition in the CE pack, as `behavior:` args |
| Bukkit listeners | Stateful gameplay (drunkenness, crab-trap ticking, projectiles) | The addon's `config.yml` |
| CraftEngine **item tags** | Borrowing a FarmersDelight behaviour | Add the FD tag to the item's `settings: tags:` (e.g. `farmersdelight:milk` to reuse the milk cleanse, `farmersdelight:compost_accelerant` to feed rich-soil compost) |
| **Special-recipe cards** | Explaining a non-recipe mechanic in the FD recipe book | Registered in code via `FarmersDelightApi.registerSpecialRecipe`; text lives in the addon's resource-pack lang JSON |

### Where an addon keeps its files

```
plugins/<Addon>/config.yml                         server-tuning knobs (this page documents them)
plugins/<Addon>/lang/*.yml                         console / log text
plugins/CraftEngine/resources/<namespace>/…        the addon's CE blocks / items / recipes / loot
```

`<namespace>` is the addon's id: `brewinandchewin`, `endsdelight`, `expandeddelight`, `crabbersdelight`,
`barbequesdelight`. Like FarmersDelight's own CE files, these are **pure data with no comments** — the
explanation lives here or in the [server guide](../server-guide/en/block-behaviors.md).

All CE addons also expose a `craftengine-resources.auto-completion` switch in their `config.yml`: on first
install the pack is written unconditionally; afterwards this controls whether missing files are topped up
(safe to leave on, or turn off if you hand-manage the pack).

***

## Brewin' And Chewin' (BAC)

*Fermentation addon — a port of Brewin' And Chewin' by ProbablyEyes (owner), Umpaz, MerchantCalico,
RaymondBlaze and Farcr. Namespace `brewinandchewin`. Soft-depends on BreweryX.*

**Adds:** the **Keg** (a fermenting block that turns ingredients into alcoholic fluids over time), fluid
**pouring** (draw a fluid into a bottle), a coaster, cheeses that **age in the keg** (ferment → cheese
fluid → pour + ripening), the **ice crate**, and four custom drink effects.

**Registered behaviours:**

* **Booze effects** — Tipsy, Sweet Heart, Raging and Intoxication are declared on the drink items as CE
  `on: consume` functions (`brewinandchewin:booze` / `brewinandchewin:tipsy`), backed by listeners for the
  stateful parts. They render through FarmersDelight's **buff bossbar** and are exposed as
  `%farmersdelight_buff_brewinandchewin_*%` PAPI placeholders.
* **BreweryX soft-compat** — when BreweryX is present, it takes over the drunkenness visuals and BAC feeds
  its tipsy amount into BreweryX's system.
* **Cheese aging cards** — `flaxen_cheese_aging` / `scarlet_cheese_aging` appear in the recipe book.

**Config (`config.yml`):** `coaster`, `keg`, `temperature`, `booze-effects`, `raging`, `sweet-heart`,
`numbed-hearts`, `tipsy`, `bossbar`, `craftengine-resources`, `language`.

***

## End's Delight (ED)

*End-themed cuisine — a port of End's Delight by FoggyHillside (MIT, `Copyright (c) 2022 FoggyHillside`).
Namespace `endsdelight`.*

**Adds:** End foods and drinks, the **End stove**, feast blocks, the two-part **dragon leg** (a bed-style
paired block), End knives, and mob drops from End creatures.

**Registered behaviours:**

* **Teleport foods** — eating certain End dishes teleports the player: gristle foods fling you upward,
  chorus dishes blink you like a chorus fruit. Implemented as the CE `endsdelight:ender_teleport` consume
  function (chorus-flower tea also uses the built-in `remove_potion_effect` function to strip Levitation).
* **Dragon-tooth knife** — deals **3.5× damage** to End creatures (endermen, shulkers, endermites and,
  configurably, the Ender Dragon).
* **Mob drops** — knife-gated and dragon drops, config-driven.
* **Cards** — `ender_teleport_foods` and `dragon_tooth_knife` appear in the recipe book.

**Config (`config.yml`):** `mob-drops` (with `knife` and `ender-dragon` sub-sections), `dragon-tooth-knife`,
`craftengine-resources`, `language`.

***

## Expanded Delight

*Additional crops & cooking — a port of Expanded Delight by ianm1647 (the upstream author). Namespace
`expandeddelight`.*

> **Scope:** this project has **no port repository for Expanded Delight in the current workspace** — the only
> trace is a scaffold under `Reference/` — so it is **not part of the project's current maintenance scope**.
> Nothing here is a licence claim for it, and the notes below describe the intended port (content mapping and
> config shape) rather than a shipped addon.

**Adds:** extra crops, foods, a workstation and cheese content. The world-generation parts of the original
mod are intentionally omitted (a plugin cannot add worldgen); everything craftable/plantable is ported.

**Config (`config.yml`):** `food-effects`, `craftengine-resources`, `language`.

***

## Crabber's Delight (CD)

*Seafood & crabbing — a port of Crabber's Delight by AlabasterLeking (MIT, declared in the upstream build
metadata only). Namespace `crabbersdelight`. Soft-depends on CustomFishing.*

**Adds:** the **crab trap** (baited block that catches loot on a timer), the **worm bin** (composts items
into worms/bait), fishing gear, a large seafood item set, the **coconut** (a falling block with an
on-land effect), collectible **notes / message-in-a-bottle**, and cutting-board recipes with drop chances.

**Registered behaviours:**

* **Crab trap** — consumes bait on an interval and rolls a bait-specific loot table. Cards for each bait
  tier (`worm → cod`, `cod → crab`, etc.) appear in the recipe book.
* **Worm bin** — feeds items tagged `farmersdelight:compost_accelerant`; a hopper-fed variant is throttled.
* **Item-tag reuse** — `coconut_milk` carries `farmersdelight:milk` to borrow FD's single-effect cleanse.
* **Cutting-board chance recipes** — registered through the FD chance API (`registerCuttingBoardRecipeWithChances`).
* **Cards** — crab-trap loot (per bait), worm bin and tackle-box bait descriptions.

**Config (`config.yml`):** `crab-trap`, `automatic-lure`, `barbed-lure`, `tackle-box`, `notes`,
`fish-plaque`, `fishing-gear-enchants`, `craftengine-resources`, `language`. Villager and wandering
trades live in the separate editable `trades.yml` file. This addon is not released yet, so no legacy
configuration migration or backup is provided.

***

## Barbeque's Delight (BBQD)

*Grilled skewers — a port of Barbeque's Delight by MaoMao and lcy0x1 (MIT by the upstream repository
`LICENSE`; its `mods.toml` says LGPL-2.1 — see the licence notes above,
[Modrinth](https://modrinth.com/mod/rtu7uERF)). Namespace `barbequesdelight`.*

**Adds:** the **Grill** (cook two skewers at once with a mid-point flip, or they burn), the **ingredients
basin** (a skewer-assembly station), a display **tray**, and **seasonings** (NBT item variants that flavour
a grilled skewer).

**Registered behaviours:**

* **Grill** — two skewer slots with a flip timer; missing the flip burns the skewer.
* **Seasonings** — right-click a grilled skewer at the grill with a seasoning to apply an effect
  (cumin heals, chilli ignites but refills hunger, honey-mustard halves duration/doubles chance, etc.).
  A `seasonings` card in the recipe book documents all six.

**Config (`config.yml`):** `grill`, `ingredients-basin`, `tray`, `seasonings`, `seasoning-anvil-enchants`,
`performance`, `craftengine-resources`, `language`.

Villager trades live in the separate editable `trades.yml` (same schema as Crabber's Delight): an offer is
one entry in the `villager` or `wandering` list, and deleting an entry withdraws that offer on the next
reload.

***

## Villagers' Delight (VD)

*Farmer-villager AI — huidu's own plugin. Namespace `villagersdelight`.*

Unlike the other addons, VillagersDelight is **NMS-based, not a CraftEngine pack** — it ships one jar per
Minecraft version and teaches **farmer villagers to recognise and farm CraftEngine crops** (FarmersDelight
crops, rich-soil farmland, and any crop you declare) while preserving all vanilla / Purpur / third-party
villager AI.

**Registered behaviours:**

* **Farming** — villagers harvest and replant configured CE crops on vanilla farmland, `extra-soils`, or a
  crop's required fluid (rice on water). Harvest drops are configurable per crop.
* **Pickup** — villagers pick up the configured seeds and foods. VD injects each CE item's base material
  into the vanilla `villager_picks_up` tag (via a per-world data pack) and filters out the vanilla
  look-alikes so a villager never grabs a plain item that merely shares that base material.
* **Sharing** — villagers throw surplus food to a *nearby villager that actually lacks it*. Which items
  count as food, and how much each is worth, comes from the `villager-ai.food.points` table.
* **Composting** — `compost-items` are added to the villager's composter work-list.

**Config (`config.yml`):**

| Key | Purpose |
| --- | ------- |
| `language` | Language file VD reads for its own console messages (`lang/en_us.yml`, `lang/zh_cn.yml`). |
| `debug` | Verbose console logging for the AI tweaks. |
| `custom-crops.enabled` | Master switch for the CraftEngine crop support. |
| `crops` | Each farmable crop → its `seed`, optional extra `soils`, `harvest-mode` (`break` / `pick` / `tall`), `water` fluid + `water-source-only`, and `plant-block`. |
| `disabled-crops` | Blacklist layered on top of `crops`. |
| `extra-soils` | Soils **every** listed crop may be planted on (vanilla farmland is always allowed). Ships with `farmersdelight:rich_soil_farmland`. |
| `harvest-drops` | Per-crop drop list (`item:count`). |
| `pickup` | `enabled`, `use-datapack-tag` (widen the vanilla `villager_picks_up` tag instead of injecting the behaviour), plus `crops` (leave empty to auto-detect from configured seeds) and `foods`. |
| `compost-items` | Items villagers may compost into bone meal. |
| `villager-ai.food` | `enabled`, `check-chance`, `protect-custom-seeds`, `minimum-kept-seeds`, and the `points` table that decides which food a villager wants (and how much it is worth). Food a villager holds is also what it shares with a nearby villager that lacks it. |
| `villager-ai.compost` | `max-items-per-work`, `minimum-kept-per-item`, `default-chance`. |
| `villager-ai.bonemeal` | `retry-delay-ticks`, `work-duration-ticks`. |
| `villager-ai.farm` | `retarget-delay-ticks`, `stop-cooldown-ticks`, `work-duration-ticks`. |

> **Config hygiene:** the shipped `config.yml` uses the `extra-soils:` key. An older deployed config that
> still uses `rich-soil-blocks:` is ignored by the loader — redeploy the current file so rich-soil farmland
> is recognised as a global soil.
