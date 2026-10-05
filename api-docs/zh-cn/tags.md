---
icon: tags
---

[English](../en/tags.md)

# 标签与可被小刀挖掘的方块

包：`com.huidu.farmersdelight.api.tag`

两个类：`FarmersDelightTags` —— 本插件使用的标签 id，都是普通 `String` 常量；`KnifeMineableBlocks` ——
对 CraftEngine 方块定义所声明标签的只读查询。

本页涉及的类型：

| 类型 | 形态 | `@ApiStatus` |
| --- | --- | --- |
| `KnifeMineableBlocks` | 静态门面，私有构造 | `@NonExtendable` |
| `FarmersDelightTags` | 静态常量容器，私有构造 | — |

## 查询"可被小刀挖掘"

```java
public static boolean isKnifeMineable(Key blockId);
```

当 `blockId` 对应的 CraftEngine 方块定义声明了 `farmersdelight:mineable/knife`
（`FarmersDelightTags.BLOCK_MINEABLE_WITH_KNIFE`）或通用标签 `c:mineable/knife`
（`FarmersDelightTags.COMMON_MINEABLE_WITH_KNIFE`）时返回 true。两个名字都算，因此按本插件家族写的包与
按通用命名空间写的包会得到同样的答案。

### 为什么入参是 CraftEngine 的 `Key` 而不是 `Block`

参数是方块的 **CraftEngine id**，不是 Bukkit 的 `Block`。把 `Block` 解析成 CraftEngine id 必须读世界里的
自定义方块状态 —— 那是区域绑定的读，而且 Bukkit 对象本身无法跨区域传递。改传 id 之后，这就成了一个
**纯定义查询**：只读 CraftEngine 的内存注册表，绝不接触世界、区块、方块实体或方块状态，因此**任何线程、
任何区域都可以调用**。

### 契约

* `null` id 返回 `false`。
* 该 id 没有 CraftEngine 方块定义时返回 `false`。
* 任何查询失败都返回 `false`：`RuntimeException` 或 `LinkageError`（CraftEngine 还没起来、定义正处于重载
  窗口、本服务器加载不了某个 CraftEngine 类）都被吞掉。**不会向调用方抛任何异常** —— "未声明"是安全的
  答案。
* 定义没有声明任何标签、或只声明了别的标签时返回 `false`。
* 不做世界访问、不做调度、调用之间不保留任何状态。

### 它不做什么

**它只提供数据。** 有三点后果需要记住：

* **它不影响挖掘速度。** 破坏速度由原版的 `minecraft:mineable/*` 标签加上手持物品的 digger 规则决定，
  而 CraftEngine 自己的工具判定走定义的 `requireCorrectTool()` / `requiredBreakPower()` /
  `isCorrectTool(..)` —— 这两套都不读方块声明的 `tags`。需要工具要求来门控什么时，请在包里用
  `require_correct_tool`。
* **它当前没有被接进任何行为。** 没有任何代码在破坏方块时调用它；掉落路径仍然是包里的 `match_item` 规则。
* **在 Java 里把掉落接到它上面会导致双掉** —— 对每个包里已经有这类规则的方块都会掉两份。所以接线之前，
  必须已经能判断"该方块是否已有 CraftEngine 掉落规则"，而当前 api 里没有这样的判据，因此不要接。

把它当作给附属和本插件内部判定用的**只读事实查询**。

## 示例

```java
import com.huidu.farmersdelight.api.tag.KnifeMineableBlocks;
import net.momirealms.craftengine.core.util.Key;

// 你手里已经有 CraftEngine 方块 id —— 来自你自己的包或配置。
if (KnifeMineableBlocks.isKnifeMineable(Key.of("myaddon:crate"))) {
    // 该方块在包里声明了可被小刀挖掘。掉落行为仍然是包里的 `match_item` 规则，
    // 不要在上面再加一份 Java 掉落。
}
```

传入你已经持有的 id 即可；从 `FarmersDelightBlocks.blockIdOf(block)` 的返回值构造 `Key` 会在别的区域上做一次区域绑定的世界读：
这正是本 api 避开的做法。

## 标签常量

`FarmersDelightTags` 收集了本插件使用的标签 id，附属可以直接引用，不必重敲字符串。它们都是普通
`String` 常量：`farmersdelight:...` 是本插件自己的标签，`c:...` 是它同时认可的通用标签。

```java
// 方块标签
public static final String BLOCK_HEAT_SOURCES         = "farmersdelight:heat_sources";
public static final String BLOCK_HEAT_CONDUCTORS      = "farmersdelight:heat_conductors";
public static final String BLOCK_MINEABLE_WITH_KNIFE  = "farmersdelight:mineable/knife";
public static final String BLOCK_MUSHROOM_COLONIES    = "farmersdelight:mushroom_colonies";
public static final String BLOCK_CABINETS             = "farmersdelight:cabinets";
public static final String BLOCK_COMPOST_ACTIVATORS   = "farmersdelight:compost_activators";
// ……另有 drops_cake_slice、tray_heat_sources、planted_from_below、feasts、pies、straw_blocks、
//     terrain、unaffected_by_rich_soil、cabinets/wooden、ropes、wild_crops、campfire_signal_smoke

// 物品标签
public static final String ITEM_KNIVES                = "farmersdelight:tools/knives";
public static final String ITEM_KNIFE_ENCHANTABLE     = "farmersdelight:enchantable/knife";
public static final String ITEM_MILK                  = "farmersdelight:milk";
public static final String ITEM_FLAT_ON_CUTTING_BOARD = "farmersdelight:flat_on_cutting_board";
// ……另有 snacks、meals、drinks、sweets、feasts、pies、serving_containers、straw_harvesters、
//     cabinets、cabinets/wooden、canvas_signs、hanging_canvas_signs、mushroom_colonies、wild_crops

// 实体类型标签
public static final String ENTITY_HORSE_FEED_TEMPTED   = "farmersdelight:horse_feed_tempted";
// ……另有 dog_food_users、horse_feed_users

// 通用（c:）标签
public static final String COMMON_MINEABLE_WITH_KNIFE  = "c:mineable/knife";
public static final String COMMON_TOOLS_KNIFE          = "c:tools/knife";
public static final String COMMON_CROPS_CABBAGE        = "c:crops/cabbage";
// ……另有 crops/tomato、crops/onion、crops/rice、storage_blocks/cabbage、storage_blocks/rice
```

完整列表就在这个类里，按前缀分组：`BLOCK_*`、`ITEM_*`、`ENTITY_*`、`COMMON_*`。常量只是 id 字符串 ——
用它不会改变标签的解析方式。

## 可用性

没有 `hasFeature` id 覆盖这个类，因此没有探测手段。它属于发布它的那些构建的 `api.**` 公开面；更老的构建
里根本没有这个类，任何引用路径都会以 `NoClassDefFoundError` 失败。要么要求一个带它的构建，要么把引用放在
"确认过类存在之后才会走到的方法体"里。和每个 api 类一样，请在**方法体内**调用（`onEnable`、监听器、命令
处理器），不要在字段初始化器或 `static final` 常量里引用。

`farmersdelight:mineable/knife` 是从方块定义的**默认状态**上读的。在 CraftEngine **26.9.1**（本构建默认
依赖所钉的版本）里，这条路径是 `BlockDefinition.defaultState().settings().tags()`。该路径不影响本 api 的
对外形态。

## 相关页面

* [方块与工作站](blocks-and-stations.md)
* [物品](items.md)
* [内容注册](content-registration.md)
