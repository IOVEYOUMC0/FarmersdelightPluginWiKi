---
icon: list-check
---

[简体中文](../zh-cn/requirements.md)

# Requirements & load order

## Server software

FarmersDelight runs on **Paper** and on **Folia**. Folia is fully supported — the plugin declares
`folia-supported: true` and routes its scheduling through Folia-aware helpers.

| Requirement | Value |
| --- | --- |
| Server | Paper or Folia (or a Paper fork: Purpur, Pufferfish, Leaf) |
| Minecraft | **1.21.5 or newer** (the 1.21.x line) |
| `api-version` | `1.21.5` |
| Java | **21 or newer** — the plugin's own floor; a Minecraft 26.x server requires **Java 25** |

The build compiles against the Paper **1.21.5** API, and the plugin's declared `api-version` is `1.21.5`, so
1.21.5 is the floor. Newer builds are fine; running below 1.21.5 is not supported.

Java 21 is a hard requirement for the plugin — the jar is compiled to the Java 21 bytecode level and will not
load on an older JRE. Use the same Java 21+ runtime CraftEngine and modern Paper already need. Keep in mind
that a Minecraft 26.x server requires **Java 25** under Paper and rejects Java 21, so run Java 25 there.

## CraftEngine is a hard dependency

FarmersDelight is a **CraftEngine port**. Every block, item, recipe and model it ships is defined as
CraftEngine content. CraftEngine is listed under `dependencies.server` in the plugin's `paper-plugin.yml`,
with `required: true`, `load: AFTER` and `join-classpath: true` (FarmersDelight resolves CraftEngine's classes
itself), which means:

- CraftEngine **must** be installed, or FarmersDelight will not enable at all;
- the server loads CraftEngine **before** FarmersDelight automatically — you do not configure load order
  yourself;
- on startup FarmersDelight waits for CraftEngine to finish parsing its items and blocks before it registers
  recipes and content.

Install a CraftEngine build compatible with your server. This repository compiles against the official Maven
**CraftEngine 26.9.1** API, and the server-side CraftEngine versions verified against a live server are
**26.8.2, 26.9 and 26.9.1**:

- 26.8.2 and newer is enough: across those three versions this plugin and every addon resolve the same
  CraftEngine classes and members (187 classes, 595 members, none missing, no unimplemented interface method),
  and the recipe, advancement and pack-section counts in the startup log are identical.
- Scope of that check: linkage (whether the classes and members exist) plus the startup result. Client-side
  behaviour was not re-verified in game on 26.8.2/26.9, and releases older than 26.8.2 were not checked.
- The shipped configs already carry the Minecraft 26.3+ world-generation definitions, but behaviour on a real
  26.3 server has not been re-verified; treat 26.3 as unverified rather than unsupported.

## Optional integrations (soft dependencies)

These are **not required**. FarmersDelight detects them at runtime and lights up the matching feature only when
the plugin is present. Everything works without any of them.

- **PlaceholderAPI** — exposes the `%farmersdelight_buff_...%` placeholders (listed in
  [Custom buffs](../../api-docs/en/buffs.md)).
- **AuraSkills** — lets cooking experience be credited as a skill instead of vanilla XP orbs
  (`experience-reward.mode` in `config.yml`).
- **UltimateAdvancementAPI** — drives the advancement tab UI.
- **Land / claim plugins** — WorldGuard, GriefPrevention, Lands, Towny, Residence, and a long list of others
  are recognised through a bundled protection layer, so station use and harvesting respect claims. No setup is
  needed beyond installing the claim plugin.

None of these change the install steps. If you have none of them, skip straight to
[Installation](install.md).

## Do not hot-reload the plugin

FarmersDelight, CraftEngine and any addon keep live references into their own class loaders (block behaviors,
per-chunk state, scheduled tasks). A `/reload` or a PlugMan-style unload tears those down mid-flight and leads
to errors. To apply changes, **stop and start the server** (`/stop`), or use the targeted
`/fd reload` / `/ce reload` commands described in [First config tweaks](first-config.md). The plugin
actively refuses hot management of itself to protect your world data.

## Next

[Installation →](install.md)

Spigot and CraftBukkit are **not** supported and never were: CraftEngine, which this plugin
hard-depends on, is itself a Paper-only plugin, and FarmersDelight uses Paper-exclusive APIs
(the data-component API, Adventure, the per-entity scheduler, Paper events). Since this
release the manifest is `paper-plugin.yml`, which makes that requirement explicit at load time
instead of failing later with a missing class.
