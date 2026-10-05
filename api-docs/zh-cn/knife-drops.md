---
icon: sword
---

[English](../en/knife-drops.md)

# 小刀掉落

包名：`com.huidu.farmersdelight.api.loot` 类：`FarmersDelightKnifeDrops` —— `final`，私有构造，`@ApiStatus.NonExtendable`。

用小刀击杀生物时的额外掉落规则注册接口，也就是 FarmersDelight 火腿、皮革、羽毛、线这些掉落背后的那套规则。 随插件发布的规则是**数据而不是 Java**：它们写在自带的 CraftEngine 资源包 `vanilla_loots.yml` 里（条目名形如 `farmersdelight:ham_from_pig`），服主在包里调整。这个门面是给需要在运行时增加规则的插件用的 —— 自带屠宰类物品的附属插件、要加条件掉落的任务插件等。

## 规则何时触发

当玩家手持匹配的工具击杀匹配类型的**成年**生物，且概率判定通过时，规则触发。

掷出的物品是**加进死亡事件的掉落列表**，而不是直接生成掉落物，因此其它插件的战利品处理会像看待任何原版掉落一样 看到它。包里定义的规则同样走死亡事件，语义一致。

## 接口

```java
public static boolean register(String entityType, String normalItemId, String burningItemId,
                               double chance, double lootingMultiplier,
                               List<String> toolItems, List<String> toolTags);

public static boolean register(String entityType, String normalItemId, String burningItemId,
                               double chance, double lootingMultiplier);

public static boolean     unregister(String entityType);
public static Set<String> entityTypesWithRules();
public static boolean     hasRule(String entityType);
```

### register

| 参数                  | 含义                                                                 |
| ------------------- | ------------------------------------------------------------------ |
| `entityType`        | Bukkit `EntityType` 名称，不区分大小写 —— `"pig"`、`"COW"` 都可以。内部会 trim 并转小写 |
| `normalItemId`      | 要掉落的物品 id，`"ns:id"` 形式。`"minecraft:air"` 或 `null` 表示不掉落            |
| `burningItemId`     | 生物着火死亡时改掉这个 id；传 `null` 则始终用 `normalItemId`                        |
| `chance`            | 基础掉落概率，0..1                                                        |
| `lootingMultiplier` | 击杀工具上每级抢夺给 `chance` 增加的量；传 0 表示忽略抢夺                                |
| `toolItems`         | 算作收割工具的物品 id 列表。为空或 `null` 时回落到插件的小刀判定                         |
| `toolTags`          | 算作收割工具的物品标签 id 列表，**不带**开头的 `#`。为空或 `null` 时回落到插件的小刀判定            |

FarmersDelight 不可用（未加载或未启用）、或 `entityType` 为 null / 空白时返回 `false`，否则返回 `true`。

**每个实体类型只有一条规则，不是列表。** 对已有规则的类型再注册，是替换。

五参数重载就是七参数版本传 `null, null`，因此规则使用插件的小刀判定：配置的小刀物品与标签，加上物品自己声明的 CraftEngine 标签 —— 与包内规则用的是同一套判定。

```java
import com.huidu.farmersdelight.api.loot.FarmersDelightKnifeDrops;

// 使用服主配置的"小刀"定义。
FarmersDelightKnifeDrops.register("rabbit", "myaddon:rabbit_cutlet", null, 0.35D, 0.10D);

// 只有本附属自己的屠宰工具能触发，烧死的生物掉熟制版本。
FarmersDelightKnifeDrops.register(
        "sheep",
        "myaddon:raw_mutton_strips",
        "myaddon:seared_mutton_strips",
        0.5D,
        0.05D,
        List.of("myaddon:cleaver"),
        List.of("myaddon:butchering_tools"));
```

### unregister

移除此前通过 `register` 添加的规则。当该实体类型上存在**由本门面注册的**规则时返回 `true`。

它只动本门面注册的规则。包内定义的掉落是独立的 CraftEngine 战利品条目，无法从这里移除——要改就走包，或自己新增一条战利品来源。

在插件禁用时调用即可。

### 查询

`entityTypesWithRules()` —— 当前**由本门面注册**了规则的实体类型名（小写）。 FarmersDelight 不可用时返回空集合。

`hasRule(entityType)` —— 该类型是否存在此类规则，不区分大小写。

这两个方法读的是门面自己的规则表，因此**不包含**包内定义的掉落（猪、牛、鸡等）。用它们来核对自己注册的条目。

## 重载存活

在此注册的规则能扛过 `/fd reload`：重载不再从配置文件重建规则表，门面注册的条目原地保留。

因此注册项能挺过服主的一次重载，你的附属插件不必为此去监听 `FarmersDelightReloadEvent`。

由于映射表背后已没有配置文件，`unregister` 立即生效；而对包内已覆盖的实体类型（猪、牛、鸡）再注册，会让该生物**同时**掉你的物品和包内的物品。

## 线程模型

`register` / `unregister` 写的是 `ConcurrentHashMap`，在 `onEnable`、重载处理或 `onDisable` 里调用都安全。查询方法 读的是当前生效的表。

从异步线程在 tick 中途注册的规则，没有文档保证能被同一 tick 内的死亡事件处理看到 —— 该表是并发容器，不会损坏，但不要假定它相对于并发击杀的可见性次序。在启动或重载时注册就不会遇到这个问题。

## 相关页面

* [物品](items.md)
* [事件](events.md)
