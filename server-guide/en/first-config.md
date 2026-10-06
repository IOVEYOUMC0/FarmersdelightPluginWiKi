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

`datapacks.tags-enabled` writes the common-item tags into the primary world's `datapacks/` folder and shares
them with every world; `enchantments.install-datapack` installs the backstabbing enchantment, and
`damage-type.install-datapack` installs the stove-burn damage type. Registry data is read once at startup, so
a change needs a server restart; `/fd reload enchant` and `/fd reload damage` reinstall the two packs. Admin
edits to already-installed files survive plugin updates. Loot injection is no longer a switch — it is part of
the FarmersDelight CraftEngine pack, so edit `vanilla_loots.yml` and run `/ce reload all`.

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

## Next

[Verifying behaviours loaded →](verifying.md)
