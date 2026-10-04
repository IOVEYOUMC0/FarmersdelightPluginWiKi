
[简体中文](../zh-cn/content-registration.md)
# Content registration

Package: `com.huidu.farmersdelight.api.registry`

`ContentRegistration` lets your plugin register its own CraftEngine **content types** — block behaviors, item
behaviors, common functions, common conditions and loot functions — under your own namespace, so your
CraftEngine pack can name them from YAML the same way FarmersDelight's own pack does.

Types covered here:

| Type | Kind | `@ApiStatus` |
| --- | --- | --- |
| `ContentRegistration` | static facade, private constructor | `@NonExtendable` |
| `ContentRegistration.Kind` | enum, 5 constants | — |

The class has no `@ApiStatus.Experimental`: the five registration methods, `isRegistered`, `registeredIds` and
`unregister` are a published surface. Everything else on it is `@ApiStatus.Internal` (see the last sections).

## What you can register

```java
public static void registerBlockBehavior(Key id, BlockBehaviorFactory<?> factory);
public static void registerItemBehavior(Key id, ItemBehaviorFactory<?> factory);
public static <T extends Function<Context>>  void registerFunction(Key id, FunctionFactory<Context, T> factory);
public static <T extends Condition<Context>> void registerCondition(Key id, ConditionFactory<Context, T> factory);
public static void registerLootFunction(Key id, LootFunctionFactory<?> factory);
```

The factory parameters are CraftEngine's own types, so implementing them means compiling against CraftEngine —
the same is true of the other helpers that take a CraftEngine `Key` or wrapper. The
`Kind` enum names the five registries (`BLOCK_BEHAVIOR`, `ITEM_BEHAVIOR`, `FUNCTION`, `CONDITION`,
`LOOT_FUNCTION`), and `Kind.label()` returns the human-readable label FarmersDelight uses in its messages and
in the late-registration warning.

The registry a type lands in is the same one FarmersDelight's own content uses, so a pack file can reference
your type anywhere CraftEngine accepts that kind of entry, with no registration order to worry about.

## Ids, namespaces and conflicts

An id must be **namespaced** — `myplugin:my_type`, not `my_type`:

* a blank namespace or a blank value throws `IllegalArgumentException`;
* the `minecraft` and `farmersdelight` namespaces are **reserved** and also throw `IllegalArgumentException`.
  A pack that could override FarmersDelight's own type ids would change behaviour for every other pack on the
  server, so the facade refuses to be the way that happens — register under your own plugin's namespace;
* a `null` id or factory throws `NullPointerException`.

A duplicate is **rejected**, never silently replaced. There are two cases, and they throw different messages:

* the same id, in the same kind, registered twice by you — `IllegalStateException` naming the kind;
* an id CraftEngine already knows — FarmersDelight's own registration or another plugin's —
  `IllegalStateException` saying it is "already registered with CraftEngine". Your factory is not applied at
  all in that case: the existing type keeps its behaviour.

Kinds have **separate registries**, so the same id may name a function and a condition at once. Only a
duplicate inside one kind is a conflict. `registeredIds()` reflects that: it returns every id currently
registered through this class, deduplicated across kinds and in unspecified order.

## Querying and withdrawing

```java
public static boolean isRegistered(Key id);      // any kind under this id
public static Set<Key> registeredIds();          // every id, deduplicated, unmodifiable
public static boolean unregister(Key id);        // returns whether anything was removed
```

`unregister` also takes a `null` id and returns `false` for it. It has a hard limit that comes from
CraftEngine, not from FarmersDelight:

**CraftEngine has no API to remove a registered type.** `unregister` stops this facade from re-applying the
entry and forgets it, but the type stays registered inside CraftEngine for the rest of the JVM run. Content
that already uses the id keeps working, and — as long as CraftEngine still holds the type, which it does for
the remainder of the run — the id **cannot be registered again**: a replacement factory would never be
consulted, so a later registration of the same id throws `IllegalStateException` with the "unregistered
earlier" reason. Use a **new id** instead of withdrawing and re-registering.

## Registering late

Registering after CraftEngine has finished loading its content still succeeds. The entry is applied, but
**content that already exists only picks the type up after CraftEngine re-reads its packs** — a `/ce reload`
(or a restart). That case is reported on the console through the `plugin.content_registration_late` key, with
the id and kind, so an operator sees why a type appears to be ignored instead of guessing.

Register as early as your addon can to avoid the warning: once CraftEngine reports its content as loaded, the
warning is the only feedback, and the type does not reach existing content until the next `/ce reload`. If your
addon also responds to CraftEngine's own reload event, re-registering an id you already registered is a
conflict — guard it with `isRegistered(id)` or register once.

## Replay is defensive, not a repair step

FarmersDelight's own registration pass calls `ContentRegistration.apply(Kind)` for each kind. That pass
re-applies only entries CraftEngine no longer reports as registered, and it is idempotent — a second pass over
an intact registry applies nothing.

The CraftEngine versions this build targets keep their built-in type registries for the whole class-loader
lifetime, so **a CraftEngine reload does not drop external registrations in the first place**: the replay is
defence in depth, not a step a reload depends on. Do not design around a CraftEngine build that rebuilds its
registries per reload, and do not read the replay as a promise that a type a CraftEngine build actually
discarded can be restored.

## Internal members, and where to call this from

`Bridge`, `apply(Kind)`, `installBridge(Bridge)` and `resetForTests()` are `@ApiStatus.Internal`. They exist
so the bookkeeping and its validation can be tested without a running CraftEngine; production code never calls
them, and addons must not either.

Like every other api class, call these methods **from inside a method body** — your `onEnable`, a listener, a
command handler. Do not reference `ContentRegistration` from an addon field initializer, a static block or a
`static final` constant: that makes the class load during your addon's class initialization, and an addon runs
under its own class loader, where a failure there (`NoClassDefFoundError`) aborts the initializer and silently
leaves whatever you were wiring disabled. See [Version compatibility helpers](compat-utilities.md) for the same
rule on `CompatAttributes`.

## Availability

No `hasFeature` id covers this class, so there is no probe for it. It is part of the published `api.**`
surface of the builds that ship it, and an older build simply lacks the class — so any addon path that
references it fails with `NoClassDefFoundError`. Either require a build that has it, or keep the reference
inside a method body you only reach after checking that the class is there.

## Example

```java
import com.huidu.farmersdelight.api.registry.ContentRegistration;
import net.momirealms.craftengine.core.util.Key;

// inside onEnable(), i.e. inside a method body
ContentRegistration.registerBlockBehavior(Key.of("myaddon:custom_keg"), MyKegBehaviorFactory.INSTANCE);
ContentRegistration.registerLootFunction(Key.of("myaddon:extra_drop"), MyExtraDropFunction.FACTORY);
```

A pack file can then use `myaddon:custom_keg` as a block behavior type and `myaddon:extra_drop` as a loot
function, exactly as it would use a `farmersdelight:*` type.

## Related pages

* [Getting started](getting-started.md)
* [Blocks and stations](blocks-and-stations.md)
* [The recipe package](recipes-overview.md)
* [Version compatibility helpers](compat-utilities.md)
