---
icon: code-branch
---

[English](../en/compat-utilities.md)

# 版本兼容工具

`com.huidu.farmersdelight.api.util` 里的小工具类存在的意义只有一个：让附属用同一个编译产物跑在全部受支持的 Minecraft 版本上。`CompatAttributes` 与 `CompatItemMeta` 抹平 Bukkit API 变更；`TooltipUtils` 与 `CeItemInterop` 处理 CraftEngine 自己的物品包装类。

支持下限已经是 1.21.5，所以两个兼容垫片在全部受支持的服务端上都能解析成功。保留它们的理由有二：它们是已发布的 API；它们仍然吸收属性注册表改名，以及未来 `setItemModel` 被移除的可能。

同包下的 `DebugToolExtension` 与 `DebugToolRegistry` 见[调试工具](debug-tools.md)，`PluginManagerGuard` 见 [快速上手](getting-started.md)。

这些类由 FarmersDelight 提供，附属直接使用即可。

## CompatAttributes

`@ApiStatus.Internal`、`@ApiStatus.NonExtendable`，final，两个公开常量。

```java
public static final Attribute MAX_HEALTH;
public static final Attribute ATTACK_SPEED;
```

Minecraft 改过属性注册表的命名：旧服务端的枚举常量是 `GENERIC_MAX_HEALTH`、注册表键是 `generic.max_health`，新服务端 改成了 `MAX_HEALTH` 和 `max_health`。代码里直接写死任何一边的枚举常量，都会导致这个类在另一边加载失败。

解析顺序是：先按新写法、再按旧写法查 `Registry.ATTRIBUTE`，都拿不到再退回反射取字段。全程不直接引用任何版本特定的 枚举常量，所以在所有受支持的版本上都能正常加载。

**当运行的服务端两种写法都没有时，常量为 null。** 传给 `getAttribute` 之前必须判空。BAC 的真实写法：

```java
AttributeInstance maxHealthAttr = player.getAttribute(CompatAttributes.MAX_HEALTH);
if (maxHealthAttr == null) return;
```

**`@ApiStatus.Internal` 在这里不是装饰，请只在方法体内引用这些常量。** 它们是静态字段，一旦你在自己的字段初始化、静态块或 `static final` 常量里写到其中一个，这个类就会在**你自己的附属**类初始化期被加载。附属跑在自己的类加载器下，那里一旦失败（`NoClassDefFoundError`），初始化会被中止，你正在装配的东西会被静默禁用 —— 插件能加载，功能却永远不会生效。写在方法体里，这次加载就落在他处理得了的路径上；而对这么小的一个查询来说，在附属里自带一份拷贝仍然更稳妥。

## CompatItemMeta

`@ApiStatus.NonExtendable`，final，两个静态方法。

| 方法                                               | 行为                                         |
| ------------------------------------------------ | ------------------------------------------ |
| `isSupported()`                                  | 当前服务端有 `ItemMeta.setItemModel` 时返回 `true`。 |
| `setItemModel(ItemMeta meta, NamespacedKey key)` | 应用 `item_model` 组件，或者什么都不做。                |

`ItemMeta.setItemModel` 从 Minecraft 1.21.4 起就存在，早于支持下限，所以反射查找（只解析一次，缓存在静态字段里）在全部受支持的服务端上都会成功。保留这个垫片，是为了让按旧 api jar 编译的附属继续可用，也为了在某个分支移除该方法时降级而不是抛异常。

`setItemModel` 在三种情况下会**静默地什么都不做**：方法不可用、`meta` 为 null、`key` 为 null。它还会吞掉 `ReflectiveOperationException`。它不抛异常也不报告失败，而且没有返回值，所以你无法从调用结果判断模型到底有没有设上。

BAC 的真实调用只对 key 的解析做了保护，版本差异完全交给这个静默 no-op 吸收，同时独立设置 custom model data，让旧版 服务端也有东西可渲染：

```java
if (customModelData != null) {
    meta.setCustomModelData(customModelData);
}
if (itemModel != null && !itemModel.isBlank()) {
    NamespacedKey modelKey = NamespacedKey.fromString(itemModel.toLowerCase(Locale.ROOT));
    if (modelKey != null) {
        CompatItemMeta.setItemModel(meta, modelKey);
    }
}
```

`isSupported()` 是留给"需要切到另一套完全不同的降级方案"的场景。目前没有任何调用方用它——所有真实调用点都直接依赖那个 静默 no-op。

## CeItemInterop

`@ApiStatus.NonExtendable`，final，三个静态转换方法，负责 CraftEngine 的 `net.momirealms.craftengine.core.item.Item` 包装类与 Bukkit 的 `ItemStack` 之间的互转。插件的容器与配方代码对每一个存入的物品都要做这些转换，所以它们集中在这里，而不是在每个调用点各抄一份。

| 方法                                               | 行为                                                              |
| ------------------------------------------------ | --------------------------------------------------------------- |
| `asBukkitStack(Item item)`                       | 返回对应的 Bukkit stack；`item` 为 `null` **或空**时返回 `null`；不看数量         |
| `toBukkitStack(Item item)`                       | 给已自行判空的调用方使用 —— **没有空物品守卫**                                     |
| `normalize(Item item)`                           | 返回该物品的防御性副本，或 `Item.empty()`                                     |

`asBukkitStack` 是带守卫的那个：`null` 或空物品返回 `null`，其余交给 CraftEngine 自己的 `ItemStackUtils.getBukkitStack`。这里不会去解析 CraftEngine 是否就绪，所以可能在 CraftEngine 就绪前被调用的调用方仍需保留自己的就绪守卫；转换内部抛出的异常也会原样传给调用方，不会被吞掉。

`toBukkitStack` 是同一个转换、但**去掉**了空物品守卫，给会自己检查转换结果、或自带回退方案的调用方使用。`null` 物品仍然返回 `null`；其余物品原样交给 CraftEngine —— 包括空物品，它会被转成空的 `ItemStack` 而不是 `null`。调用点要按「拿到空结果时自己怎么办」来选其中一个。

`normalize` 返回的是可以直接留存的物品：参数为 `null`、空物品、或数量不是正数时返回 `Item.empty()`，否则返回**副本**（`copyWithCount`，数量相同）而不是你传进来的那个实例。正因为是副本，你留存结果的同时调用方继续改自己那份也不会互相影响。

```java
Item stored = CeItemInterop.normalize(incoming);   // 永不为 null；有内容时是副本
ItemStack bukkit = CeItemInterop.asBukkitStack(stored);   // 空物品时为 null
```

## TooltipUtils

final，无 `@ApiStatus` 注解，一个静态方法：

```java
public static void hideDurabilityLine(Item wrapped)
```

注意参数类型：这是 CraftEngine 的 `net.momirealms.craftengine.core.item.Item` 包装类，**不是** Bukkit 的 `ItemStack`。需要用到它的调用方本来就已经在 CraftEngine 那一侧了，所以签名没有粉饰这一点。若需使用，先用 `BukkitItemManager.instance().wrap(stack)` 包装。

这个方法把 `minecraft:damage` 和 `minecraft:max_damage` 加进 `minecraft:tooltip_display.hidden_components`，并且是与已有条目**合并**而不是覆盖。当物品的耐久值编码的不是工具磨损 而是别的东西——进度条、份数——就用它，免得玩家看到一行毫无意义的 "Durability: X / Y"。

改动作用在包装类上，会写回你包装的那个 stack。调用方的原 stack 不能被动到时，先 clone。FDAddonTemplate 完整演示了把 容量编码进耐久条的整套做法：

```java
ItemStack copy = stack.clone();
Item wrapped = BukkitItemManager.instance().wrap(copy);
wrapped.maxDamage(capacity);
wrapped.damage(Math.max(1, capacity - Math.min(capacity, amount)));
TooltipUtils.hideDurabilityLine(wrapped);
return copy;
```

`Math.max(1, ...)` 是有用的：原版在 damage 为 0 时会把耐久条整个隐藏，不这样写的话，装满的容器反而没有耐久条。

## 相关页面

* [快速上手](getting-started.md)
* [内容注册](content-registration.md)
* [调试工具](debug-tools.md)
* [物品](items.md)
