---
icon: box
---

[简体中文](../zh-cn/resource-pack.md)

# Resource pack

## CraftEngine serves the pack, not FarmersDelight

FarmersDelight ships the models, textures and lang entries for its blocks and items, but it does **not** host
or send a resource pack. It writes those files into `plugins/CraftEngine/resources/farmersdelight/`, and
**CraftEngine** assembles them into the pack and delivers it to players.

That means:

- Everything about **generating, hosting and sending** the pack is configured in CraftEngine, and documented
  in **[CraftEngine's own documentation](https://mo-mi.gitbook.io/xiaomomi-plugins/)**. Follow it there — there
  is nothing FarmersDelight-specific to configure.
- When you change a FarmersDelight resource file, you re-generate the pack the CraftEngine way. Use
  `/ce reload all` (config **and** pack) or `/ce reload pack` (pack only) — plain `/ce reload` reloads only
  configuration and does **not** rebuild the pack. This is exactly how you would refresh any other CraftEngine
  content.

## The two failure modes you will actually hit

### 1. Purple-and-black missing textures

If custom blocks and items appear as the classic purple/black checkerboard, the **server loaded fine but the
client is not using the current pack**. Almost always one of:

- you added or edited resources but did not regenerate the pack — run **`/ce reload all`** (plain
  `/ce reload` does not rebuild the pack);
- the player has not accepted / downloaded the pack, or their client has "Server Resource Packs" set to
  disabled — check the client's server settings;
- a cached older pack is stuck — have the player rejoin, or clear their pack cache.

Missing textures are a **pack delivery** problem. The plugin itself is fine — you can confirm by checking that
`/ce item give <player> farmersdelight:cooking_pot` still hands over an item with the right *name* (the name
comes from lang data, the texture from the pack).

### 2. Pack hosting / players never receive it

If players join and are never prompted for a pack, or the download fails, this is CraftEngine's pack-hosting
configuration — self-hosted, external host, or the built-in host, depending on how you set CraftEngine up.
This is entirely a CraftEngine concern; see CraftEngine's documentation on pack hosting. FarmersDelight has no
setting that affects delivery.

## Tooltip item icons

A cooking pot's tooltip shows the icon of the meal stored in it, and a keg's shows the icon of its drink. Those
icons are **image glyphs** declared in a pack's `configuration/*.yml` `images:` section: a font glyph whose texture
is the item's own 16x16 texture. The shipped content uses:

| Line | Id | Declared in |
| --- | --- | --- |
| Cooking pot meal | `<namespace>:meal_<item>` | the pack that ships the meal item |
| Cooking pot meal that is a vanilla item (`beetroot_soup`, `mushroom_stew`, `rabbit_stew`) | `farmersdelight:meal_<material>` | the FarmersDelight pack |
| Keg drink (Brewin' And Chewin) | `brewinandchewin:icon_<drink>` | the Brewin' And Chewin pack |

Every entry uses the same two metrics, which describe the tooltip line rather than the texture:

- `height: 16` keeps the glyph 1:1 with the 16x16 item texture;
- `ascent: 8` places it so it does not overlap the line above; from 9 upwards it covers that line;
- the tooltip adds one empty line after the icon, because the lower half of the glyph needs a line to render in.

An entry may point at a vanilla texture instead of shipping one, for example `file: minecraft:item/mushroom_stew.png`.
There `height` is **required**, because there is no PNG for CraftEngine to measure, and a wrong path is silent: the
reload succeeds and the client renders the checkerboard inside that one tooltip line. So after renaming or moving a
food texture, check that tooltip in game. Icons follow the item ids, so deleting an item leaves entries behind that
are simply never looked up.

## When to re-run `/ce reload all`

Run it whenever you touch anything under `plugins/CraftEngine/resources/farmersdelight/` — a texture swap, a
model tweak, a block or recipe edit. `/ce reload all` re-reads the configuration and rebuilds the pack so the
next join (or the next in-game pack refresh) picks up the change. (Plain `/ce reload` reloads configuration
only and skips the pack rebuild, so a texture or model change would not reach clients.)

## Next

[First config tweaks →](first-config.md)
