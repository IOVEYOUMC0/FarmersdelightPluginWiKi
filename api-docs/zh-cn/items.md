---
icon: label
---

[English](../en/items.md)

# 物品

包名：`com.huidu.farmersdelight.api.item` 类：`FarmersDelightItems` —— `final`，私有构造。

一组物品辅助方法，统一处理 CraftEngine 自定义物品、原版材质和物品标签的解析与匹配。签名只用 Bukkit / Adventure / `java` 类型，附属插件的编译 classpath 上不需要 CraftEngine。

该类位于受支持的 `com.huidu.farmersdelight.api.**` 附属 API 中，因此附属插件可以直接调用。

## 身份识别

```java
public static String       idOf(ItemStack item);
public static ItemStack    create(String itemId);
public static List<ItemStack> createChoices(String itemId);
public static boolean      matchesId(ItemStack item, String itemId);
public static Set<String>  idsOf(ItemStack item);
public static boolean      isCustomItem(ItemStack item);
public static String       customIdOf(ItemStack item);
```

`idOf` —— 按 CE ID → 旧 `mmoitems:TYPE:ID` → 已绑定 CE 外部来源 → 小写 `minecraft:<material>` 解析；空气或 null 返回 `null`。

`create` —— 先查 CE 自定义物品，再通过指定 `ItemSource` 构造外部物品，原版 ID 查 Bukkit 材质。来源或模板不存在时返回 `null`，不生成原版替代物。外部格式与来源绑定见[首要配置项](../../server-guide/zh-cn/first-config.md)。

`matchesId` —— CE、旧 MMOItems 和原版 ID 保留不区分大小写的比较；其他外部来源的原始 ID 区分大小写，来源名转小写。CE/外部物品只匹配其真实身份，不匹配载体的原版 ID。

`idsOf` —— 该 stack 能解析到的所有带命名空间 id：CE/外部真实 id 和 / 或原版材质 id。该辅助集合保留载体用于配置，不代表配方匹配会接受载体。空气或 null 返回空集合。

`isCustomItem` / `customIdOf` —— 是否为 CraftEngine 自定义物品，以及它的自定义 id（纯原版材质或空 stack 返回 `null`）。

真实附属插件就是这么封装的。BrewinAndChewin 的 `BrewinItems` 是一层很薄的转发，外加一个缓存：

```java
import com.huidu.farmersdelight.api.item.FarmersDelightItems;

public static String idOf(ItemStack item) {
    return FarmersDelightItems.idOf(item);
}

public static ItemStack create(String itemId) {
    return FarmersDelightItems.create(itemId);
}
```

构造一个 CraftEngine 物品会重新解析它的 MiniMessage 名称，放在逐 tick 的路径上开销是能测出来的。BrewinAndChewin 按 id 记忆已构造的 stack 并对外发克隆，重载时清空缓存，好让改过的 CraftEngine 定义生效：

```java
private static final Map<String, ItemStack> CACHE = new ConcurrentHashMap<>();

public static ItemStack createCached(String itemId) {
    if (itemId == null) {
        return null;
    }
    ItemStack base = CACHE.get(itemId);
    if (base == null) {
        base = FarmersDelightItems.create(itemId);
        if (base == null) {
            return null;
        }
        CACHE.put(itemId, base);
    }
    return base.clone();
}
```

这个模式只在真正的热路径上照抄；一次性调用直接用 `create` 就够了。

## 标签

```java
public static boolean      matchesTag(ItemStack item, String tagId);
public static Set<String>  tagIdsOf(ItemStack item);
```

`matchesTag` —— `item` 是否带有给定标签。`#ns:tag` 和 `ns:tag` 两种写法都接受，开头的 `#` 会被去掉。它会**同时** 检查 CraftEngine 自定义物品标签和原版物品标签。

`tagIdsOf` —— 返回 CE 声明和主插件/附属注册的公共标签，按物品真实身份匹配，不让自定义物品继承其载体原料标签。它不枚举原版全部物品标签；直接判断原版标签请用 `matchesTag`。空气或 null 返回空集合。

```java
if (FarmersDelightItems.matchesTag(tool, "#farmersdelight:tools/knives")) {
    // 自定义标签和同名原版标签都能命中
}
```

## 合成返还物

```java
public static ItemStack craftingRemainderOf(ItemStack item);
```

单个 `item` 留下的返还物 —— 奶桶还桶、蜂蜜瓶还玻璃瓶，或者 CraftEngine 里配置的容器返还。不留返还物时为 `null`。

解析顺序是：自定义物品先查 CraftEngine 的容器返还映射，然后是原版 `Material.getCraftingRemainingItem()`，最后是 奶桶 / 水桶 / 岩浆桶和蜂蜜瓶这几个特例。这和 FarmersDelight 自己的工作站行为一致，因此附属配方的容器返还能和厨锅 保持统一。

## 展示名

四种不同的渲染方式，选错是真会出 bug —— 比如把只对某一个玩家语言解析好的名字，写死到一件所有人都能看到的物品上。

```java
public static Component displayNameOf(ItemStack item, Player player);
public static Component translatableDisplayNameOf(ItemStack item);
public static Component translatableDisplayNameOfNoAnvilOf(ItemStack item);
public static Component serverDisplayNameOf(ItemStack item);
```

| 方法                                         | 解析方式                                                     | 铁砧改名 | 适用场景                                |
| ------------------------------------------ | -------------------------------------------------------- | ---- | ----------------------------------- |
| `displayNameOf(item, player)`              | 按这一个玩家的语言解析（`player` 可为 null）                            | 保留   | 当下发给单个观看者的文本                        |
| `translatableDisplayNameOf(item)`          | `Component.translatable(key)` + 服务端解析好的 `.fallback(...)` | 原样保留 | 写入 stack 的 lore，或发给多人的广播            |
| `translatableDisplayNameOfNoAnvilOf(item)` | 同上，但忽略铁砧改名                                               | 忽略   | 需要跟随各自客户端语言、又不能被玩家铁砧改名写死的持久 lore    |
| `serverDisplayNameOf(item)`                | 完全在服务端按服务端默认语言烘焙，绝不返回 translatable                       | 忽略   | 需要在所有客户端呈现完全相同字符、不受客户端语言与资源包状态影响的文本 |

两个 translatable 变体上的 `.fallback(...)` 很关键：资源包里缺这条 lang 键的客户端会看到服务端默认语言的可读 文本，而不是裸的 `item.ns.id` 键名。

另外注意 `translatableDisplayNameOfNoAnvilOf` 这个方法名 —— 末尾重复的 `Of` 是它真实的名字，不是笔误。

## 把名称和 lore 渲染到物品上

```java
public static void      applyDisplay(ItemStack item, String nameTemplate, List<String> loreTemplates,
                                     Player viewer, Map<String, String> placeholders);
public static ItemStack buildIcon(String itemId, String nameTemplate, List<String> loreTemplates,
                                  Player viewer, Map<String, String> placeholders);
```

`applyDisplay` 用模板设置展示名和 lore，模板经由 `FarmersDelightText` 渲染 —— 支持字符串与 `Component` 占位符、 `<l10n:>` 标签、CraftEngine 字体图片、MiniMessage 以及旧版颜色代码，并去掉名称上原版默认的斜体。`nameTemplate` 或 `loreTemplates` 传 `null` 时对应部分保持不变。`item` 为 `null`、或该 stack 没有 `ItemMeta` 时静默返回。

用它可以让你的附属插件的提示文本和 GUI 图标与 FarmersDelight 的渲染保持一致，而不是各写一套。

`buildIcon` 就是 `create(itemId)` 后接 `applyDisplay`；`itemId` 解析不到时返回 `null`。

```java
ItemStack icon = FarmersDelightItems.buildIcon(
        "farmersdelight:cooking_pot",
        "<l10n:myaddon.gui.pot_title>",
        List.of("<l10n:myaddon.gui.pot_lore>"),
        viewer,
        Map.of("count", String.valueOf(count)));
```

线程：`applyDisplay` 和 `buildIcon` 只操作你传进去的 `ItemStack`，但它们会读取 FarmersDelight 的翻译状态。请在 将来实际使用该 stack 的那个线程上调用，不要在异步任务里和 `/fd reload` 抢跑。

## 相关页面

* [文本与消息](text-and-messages.md)
* [方块与工作站](blocks-and-stations.md)
* [版本兼容工具](compat-utilities.md)

## 标签原料的显示候选

`createChoices("#namespace:tag")` 复用主插件已合并的 CE / 原版 / 附属标签，返回可构造的候选物品列表；不存在时为空。普通 ID 返回至多一个候选。用于配方展示或调试填充，真实匹配仍用 `matchesTag`，不要把首个候选写死成唯一原料。
