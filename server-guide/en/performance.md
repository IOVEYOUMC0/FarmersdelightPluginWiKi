---
icon: gauge
---

[简体中文](../zh-cn/performance.md)

# Performance diagnosis & tuning

Identify the expensive feature before changing its settings. `performance.warnings` controls logging; raising a threshold saves no computation. Default budgets do not guarantee server MSPT.

## Run a comparable profile

1. Record Paper/Folia, Minecraft, CE, FD and addon versions, player counts, loaded stations and stations actually cooking. On Folia, also record whether stations share one region or span several regions.
2. Wait for chunk recovery, recipe registration and the *Content ready* message. Observe idle stations, normal cooking and dense automation separately. Distinguish the first recipe-book open from later opens.
3. With `farmersdelight.admin`, run `/fd stats profile 600 all`, about 30 seconds at normal TPS. Use `/fd stats` afterward. Feature selectors and sample limits are documented in [Profile individual features](troubleshooting.md#profile-individual-features).
4. Use the server's spark profiler to inspect executing-thread call stacks, allocations/GC and MSPT. On Folia, inspect busy region threads as well as the global scheduler.
5. Repeat three times with the same load. Change one setting and repeat the measurements. Compare maximum time and P95, and check serving throughput, hopper pacing and displays.

Feature timings include callees and cannot simply be added together. They measure elapsed time. Network submission is not actual transmission, and region timings are not server-wide MSPT. Zero calls usually means the feature did not execute in that window.

## Profile saves and serialization

```text
/fd debugtools profile serialization 600
/fd debugtools stop
/fd stats
```

Requires a debug build and `farmersdelight.admin`; release builds can use `/fd stats profile 600 serialization`. The debug entry defaults to 600 ticks and accepts 20-12000. Only one server-wide session runs at a time, capturing executing calls across regions rather than filtering by the initiating player.

Results cover live container saves, item NBT encoding, instrumented save-lock acquisition and holder queries with snapshots disabled. Rows show calls, average, P95, maximum and total time. Save hooks cover FD, BAC, CAB and BBQD; parked-tag passthrough skips live-save timing. Encoding can also occur in other instrumented paths such as item packing; lock timings exclude an outer lock already held by the caller.

Operate normally and wait for autosave; opening a GUI does not guarantee persistence. Zero calls mean the path did not execute in the window, so extend the duration if needed. No save, chunk unload or reload is forced, and inventory contents are unchanged. `/fd stats` retains results after stopping. Nested rows are not additive MSPT and exclude whole CE chunk encoding, compression and disk writes. Use spark alongside these measurements for call stacks and GC.

## Budget effects and tradeoffs

These are fresh-install defaults; existing servers use their actual `plugins/FarmersDelight/config.yml`. Apply FD setting changes with `/fd reload config`. Replacing a plugin JAR requires a full restart.

| Config path | Default | Tuning tradeoff |
| --- | --- | --- |
| `cooking-pot.tick-budget` | 512 / 4 ticks | Fewer active pots per pass; rotation above the budget can delay responses and serving |
| `skillet.tick-budget` | 512 / 4 ticks | Fewer placed pans polled; cooking may slow above the budget |
| `stove.tick-budget` | 512 / 4 ticks | Fewer cooking stoves polled; processing and serving may be delayed |
| `performance.budgets.reload-recipe-registrations-per-tick` | 32 | Slower reload completion with less registration work per tick; 0 is treated as 1 |
| `performance.budgets.reload-visual-refreshes-per-tick` | 32 | Slower station display recovery after reload |
| `performance.budgets.startup-chunk-loads-per-tick` | 16 | Slower FD recovery from already-loaded chunks at startup; does not control every addon |
| `performance.budgets.chunk-effect-packet-budget` | 50 | Fewer particle/sound broadcasts per chunk per tick; each broadcast may reach many players, so this is not a total packet cap |
| `performance.budgets.effect-tick-interval-ticks` | 4 | Raise this to update food effects less often; bars become less smooth, while durations still decrement by the interval |
| `performance.proxy-display.view-distance` | 64 | Distant station displays disappear sooner |
| `performance.proxy-display.sync-interval-ticks` | 20 | Raise this for fewer visibility checks; display changes when approaching/leaving take longer |
| `performance.proxy-display.sync-batch-size` | 256 | Fewer displays checked per pass; a complete rotation takes longer |
| `skillet.handheld.progress-display.enabled` | true | Disabling removes periodic progress refreshes while retaining handheld cooking |

Pots and stoves advance their timers in game ticks, but insufficient budgets can still delay completion processing. Skillet cooking can also slow. Compare gameplay throughput as well as per-tick time when lowering budgets.

## Diagnose by scenario

| Scenario | Observe first | Adjustments to try |
| --- | --- | --- |
| Dense pots and hopper automation | `cooking_pot` P95/max; matching and hopper-event stacks | Pot budget, effect probabilities and viewer distance; server settings and other plugins also affect hopper pacing |
| Many handheld pans | `handheld`, `handheld_display`; model and packet allocations | Disable periodic progress display and compare; use `skillet.handheld.enabled` to disable the mechanic |
| Many station displays | Visibility-sync stacks and actual network load | Reduce view distance, raise sync interval, moderately reduce batch size |
| Slowdown after adding exact NBT recipes | Serialization, Base64 and GC stacks | Use identity/tag ingredients when full item-data constraints are unnecessary; keep NBT when required for correct matching |
| Startup or reload spikes | Registration, chunk recovery, addon recovery and pack generation separately | Lower the relevant FD budget; CE pack generation and addon recovery need their own analysis |
| Slowdown only when stations share a region | The busiest Folia region thread | Spread dense machinery and compare; locks and async tasks cannot grant ownership of Bukkit objects |

Reuse CE's registered external item sources and configure only sources you use; see [Experience and external item sources](first-config.md). Missing sources stay retryable for CE's delayed registration. Correct persistently unknown source names.

## Next

[Migration & upgrades →](migration.md)
