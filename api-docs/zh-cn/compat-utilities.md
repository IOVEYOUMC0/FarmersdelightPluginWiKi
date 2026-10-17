---
icon: code-branch
---

[English](../en/compat-utilities.md)

# 版本兼容工具

`com.huidu.farmersdelight.api.util` 提供 CE 物品互操作、可选性能计时和已发布的兼容接口。最低支持 Paper 1.21.5；新代码直接使用该版本已有的 Bukkit/Paper API。`CompatAttributes` 与 `CompatItemMeta` 保留签名供旧附属调用，内部已移除反射。

同包的 `DebugToolExtension` 与 `DebugToolRegistry` 见[调试工具](debug-tools.md)，`PluginManagerGuard` 见[快速上手](getting-started.md)。

## PerformanceProbe

`PerformanceProbe.begin(Operation)` 在对应计时未开启时返回 `null`，不读时钟或分配样本；开启时返回实现 `AutoCloseable` 的 `Sample`。Java 的 try-with-resources 允许资源为 null，异常退出也会记录耗时。Sample 单次记录，旧会话结束后不会写入新会话；同一样本由同一执行线程关闭。

`item(ItemStack)` / `items(ItemStack[])` 直接调用 CE 原有 codec；`holder(Inventory)` 固定使用 Paper 的 `getHolder(false)`。这些帮助方法不调度、不改线程归属或锁，不代表可在后台访问实时库存。附属只导入 `api.util.PerformanceProbe`，与本轮 FD 一起更新。

```java
import com.huidu.farmersdelight.api.util.PerformanceProbe;

try (PerformanceProbe.Sample ignored = PerformanceProbe.begin(PerformanceProbe.Operation.SERIALIZATION)) {
    // Keep the original serialization and error handling here.
}
```

`Operation` 包含 `SERIALIZATION`、`ITEM_SERIALIZATION`、`SERIALIZATION_LOCK`、`INVENTORY_HOLDER`。锁样本在进入 synchronized 前开始，进入后的第一行关闭，不把编码算作等待。Facade `FarmersDelightApi.get().performanceTiming(operation)` 返回绑定当前会话的 recorder，未采样/插件不可用时返回 null；不得长期缓存。使用方式见[调试工具](debug-tools.md)。

## CompatAttributes

`@ApiStatus.Internal` 的 `MAX_HEALTH` 与 `ATTACK_SPEED` 现在直接对应 Bukkit 的属性常量。新附属使用 `Attribute.MAX_HEALTH`、`Attribute.ATTACK_SPEED`，不需要注册表探测或另一份兼容工具。`getAttribute` 仍可能因实体不支持该属性而返回 null。

```java
import org.bukkit.attribute.Attribute;
import org.bukkit.attribute.AttributeInstance;

AttributeInstance maxHealth = player.getAttribute(Attribute.MAX_HEALTH);
if (maxHealth == null) return;
```

## CompatItemMeta

| 方法 | 行为 |
| --- | --- |
| `isSupported()` | 支持范围内恒为 true；保留旧签名。 |
| `setItemModel(ItemMeta meta, NamespacedKey key)` | meta/key 为 null 时不操作；其它情况直接调用 Paper API。 |

新代码直接使用 `meta.setItemModel(modelKey)`。旧兼容接口不再吞掉调用错误。自定义模型数字使用 `CustomModelDataComponent`；不再使用过时的整数 setter。

```java
import org.bukkit.inventory.meta.components.CustomModelDataComponent;
import java.util.List;

CustomModelDataComponent data = meta.getCustomModelDataComponent();
data.setFloats(List.of(customModelData.floatValue()));
meta.setCustomModelDataComponent(data);
meta.setItemModel(modelKey);
```

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

## 启动数据包工具

`DatapackSupport` 提供 `managedPack`、`startupConfiguration`、`discoverPack`、`migrateLegacyPack` 与 `archiveLegacyPack`。仅在 Paper 的 bootstrap 数据包发现阶段调用 `discoverPack`；它通过官方 `DatapackRegistrar` 首次发现并加载受管包，尊重保存的手动禁用状态。配置读取严格报错，文件写入使用临时文件替换；旧世界包复制到受管目录成功后清理，不自动备份。`archiveLegacyPack` 仍保留为显式调用工具，不参与自动迁移；已有备份不清理。

`DamageTypePack.prepare(configuration, defaults, pack, description, version)` 从伤害 YAML 生成 JSON 和元数据，`saveConfiguration(configuration, pack)` 仅在 YAML 不存在时导入旧 JSON。配置包含 `damage-types` 与 `tags` 两个映射，键为完整注册 ID，字段采用 Minecraft 数据包原名。JSON 是生成产物，改 YAML 后完整重启。FD、BAC、CAB 共用该工具，两个附属的 bootstrap 均对 FD 声明依赖。新附属请在自己的 bootstrap 完成首次注册；onEnable 注册的动态内容仍需下次启动。

这使用 Paper 的数据包发现 API，仍加载 JSON。直接注册表接口在 1.21.5 用 `freeze()`、1.21.6 起用 `compose()`；不能静态调用新版方法覆盖整个支持范围。

## 相关页面

* [快速上手](getting-started.md)
* [内容注册](content-registration.md)
* [调试工具](debug-tools.md)
* [物品](items.md)
