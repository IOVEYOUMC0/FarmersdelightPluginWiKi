---
icon: sliders
---

[简体中文](../zh-cn/first-config.md)

# First config tweaks

The defaults in `plugins/FarmersDelight/config.yml` are tuned to behave like the original mod, so a fresh
install is playable untouched. This page lists only the handful of settings **most owners change on day one**
and points at the page that covers each one in full. It does not re-document the config — the comments in the
shipped `config.yml` are the reference.

Apply changes with `/fd reload config` (or restart). Do **not** `/reload` the whole server.

## Console / log language

```yaml
language: ''
```

Empty follows the JVM/system language. Force it with `zh_cn` or `en_us`. This affects **server logs, console
text and messages with no player context** — individual players still see GUIs and chat in their own client
language. See [Troubleshooting](troubleshooting.md) for how the console language resolves.

## Buff display (bossbars)

```yaml
buff:
  enabled: true
  display:
    enabled: true
    channels:
      - bossbar
    layout-mode: stacked
    styles:
      nourishment: {color: GREEN, overlay: PROGRESS}
      comfort:     {color: BLUE,  overlay: PROGRESS}
```

Controls how the Nourishment/Comfort buffs (and any addon buffs) are shown. Switch `channels` to `actionbar`
or `tab_footer` if another plugin already owns the boss-bar area, or set `display.enabled: false` to keep the
effects but hide the bars. Full detail: [Custom buffs](../../api-docs/en/buffs.md).

## Effects: Nourishment / Comfort

```yaml
events:
  - on: consume
    functions:
      - type: farmersdelight:nourishment
        duration: 180
```

Food assignments now live in each item's CraftEngine configuration. Use `farmersdelight:nourishment` or
`farmersdelight:comfort` as an `on: consume` function; `duration` is in seconds. The built-in assignments are in
`plugins/CraftEngine/resources/farmersdelight/configuration/items.yml`. Global buff display, persistence and
Comfort healing settings remain in `plugins/FarmersDelight/config.yml`.

## Hopper interactions

```yaml
hopper-interactions:
  enabled: true
```

The master switch for the hopper ↔ station bridge (cooking pot, cutting board, skillet). If hoppers misbehave
with your other plugins, **turn this master switch off first** to isolate the problem, then re-enable and tune
each station's own `allow-hopper: true/false` in `config.yml`.

CrabbersDelight crab traps and worm bins use native hopper containers independently of the FD station switch above. A hopper pointing down from above inserts bait/compost; a hopper below extracts products. Amount and cooldown follow server settings (defaults: one item per transfer and eight ticks), including redstone locking and inventory move events. Tackle box furniture has no hopper access.

## Cooking XP and CE levelers

```yaml
experience-reward:
  mode: levelers
  levelers:
    - plugin: mmocore
      target: cooking
      multiplier: 1.0
```

Use the provider name registered by CE for `plugin` and the actual skill, profession or job ID for `target`. Target case and separators are preserved. The example requires a MMOCore profession/collection skill named `cooking`; it does not award character main-level XP. Availability follows CE's registrations. The CE 26.9.2 reference includes `auraskills`, `aureliumskills`, `ecojobs`, `ecoskills`, `jobs`, `mmocore`, `mcmmo` and `minecraft`.

| mode | Reward |
| --- | --- |
| `vanilla` (default) | Vanilla XP orbs |
| `auraskills` | The legacy `auraskills` section |
| `levelers` | The CE `levelers` list |
| `both` | Orbs + skill rewards; a generic `auraskills` entry replaces the legacy AuraSkills section |
| `none` | No rewards; the read-only settlement event still fires |

Each entry accepts fixed `amount` (overrides the multiplier), `multiplier` (recipe base × multiplier) and `chance` (0–1, or percentages above 1). The base must be finite and positive; amounts cannot be negative or infinite. Missing providers or unavailable targets never switch to a different reward.

A successful player take from the **output slot** settles all accumulated recipe batches, rather than prorating XP by the amount taken. Hand serving from the pending meal, bowl transfer, and hopper extraction do not claim XP. The ledger is persisted, then cleared and marked dirty when claimed. Removal releases unclaimed vanilla orbs before packing items, with no skill reward. Saves lacking a ledger cannot reconstruct recipe batches from remaining servings. Fixed amounts and chance apply per settlement. Addon stations using the FD XP API, including kegs, share this configuration. Cutting boards have no recipe XP base. Ordinary AuraSkills rewards use CE; legacy `raw: true` retains direct raw-XP semantics. CE's `minecraft` provider gives integer XP directly, unlike the probabilistically rounded orbs in `vanilla` mode; configuring it under `both` awards both channels.

Orbs spawn on the station's region scheduler. Skill rewards and cooking-pot/API events run on the player's entity scheduler. Reusing CE does not make a third-party plugin Folia compatible.

## External item sources

CE's registered `ItemSource` providers can supply FD ingredients, tools, containers and results: for example `itemsadder:myfoods:rice`, `nexo:rice_knife` and `mmoitems:AXE:TEST`. The first colon separates the provider name; the remainder is the original item ID. Keep its case and colons.

Sources referenced by recipes bind automatically. To recognize held items in the editor before any recipe references them, add this to FD `config.yml`:

```yaml
external-item-sources: [itemsadder, nexo, oraxen]
```

Legacy `mmoitems:TYPE:ID` recognition is always retained. Identity order is CE ID, legacy MMOItems data, bound external sources sorted by provider name, then vanilla material. External items cannot satisfy their base material's vanilla ID or tags. Tags come from CE declarations or the FD tag API; CE's `recipe.ingredient-sources` only controls CE recipes.

A missing source/template returns `null` and never creates a vanilla substitute. Editor inputs/tools match the real identity; results and data-bearing containers retain full snapshots. Existing ID-only MMOItems recipes rebuild through CE. Results are fixed templates built at recipe load, not fresh player-specific or random results for each craft. Apply source configuration with `/fd reload all`; CE reload also clears bindings and icon caches.

## Recipe discovery (locked recipe books)

```yaml
recipes:
  discovery:
    enabled: false
```

Off by default — every recipe is visible in the books immediately. Turn it on to make recipes start **locked**
in FarmersDelight's own cooking-pot / cutting-board viewers and reveal per-player as they unlock. Locking only
affects the book display; it never blocks crafting at a station. `locked-display`, `unlock-on-obtain` and
`notify` sit under the same section.

> Recipe unlock progress is keyed by **recipe id**. Renaming a recipe id resets discovery for it. See
> [Migration & upgrades](migration.md).

## Data packs shipped with the plugin

```yaml
datapacks:
  tags-enabled: true
enchantments:
  install-datapack: true
damage-type:
  install-datapack: true
```

Paper discovers the managed packs under `plugins/FarmersDelight/datapacks/` during startup. These three switches keep their meaning. Common tags come from `common-tags.yml`, backstabbing stays in `config.yml`, and stove damage definitions and tags come from `damage-types.yml`. BAC and CAB own their cardiac-arrest and falling-coconut definitions in their respective `damage-types.yml` files.

Existing world packs remain active during the migration boot, then move intact to each plugin's `datapack-backups/` directory. Legacy damage JSON is imported when no damage YAML exists; an existing YAML configuration takes precedence. The next full startup uses the plugin-managed packs. Registry IDs stay unchanged; keep IDs already used by your world.

JSON under `datapacks/` is generated output. Configuration changes require a full restart. `/fd reload enchant` and `/fd reload damage` regenerate files without making new registry entries live in the current run. FD and VD no longer request server data reloads. New addon contributions to vanilla tags still require the next startup. Keep backups until you have verified the migration.

Loot injection remains part of the CE content pack: edit `vanilla_loots.yml` and run `/ce reload all`.

## Advancements

```yaml
advancements:
  enabled: true
```

Turn the whole advancement tree off with `enabled: false`. Left on, `auto-disable-missing: true` hides
advancements whose CraftEngine content you deleted so the tree stays completable. Background:
[Migration & upgrades](migration.md) and [Advancements](../../api-docs/en/advancements.md).

## Where the deeper knobs live

Performance budgets (`performance.warnings` / `performance.budgets` / `performance.proxy-display`),
particle/sound effects, display offsets, heat sources and custom-item `container-returns` are in `config.yml`.
The per-item and per-tag board display tables are in `plugins/FarmersDelight/display-overrides.yml`
(`items` / `tags`). The straw-drop whitelist is in `plugins/FarmersDelight/drops.yml`, while the knife mob
drops are **pack data**: they live in the CraftEngine pack file `vanilla_loots.yml` (entries named
`farmersdelight:ham_from_pig` and the like), tuned alongside the rest of the pack's loot. Villager and
wandering-trader offers are in
`plugins/FarmersDelight/world-data.yml`; deleting an offer there disables it. Composting, furnace-fuel values,
pet food and food-buff assignments are in the CraftEngine item configuration. CraftEngine files are
comment-free; their field reference is in [Block behavior configuration](block-behaviors.md). You will rarely
need them on day one.

See [Performance diagnosis & tuning](performance.md) for the current budget defaults and tuning tradeoffs.

## Next

[Verifying behaviours loaded →](verifying.md)
