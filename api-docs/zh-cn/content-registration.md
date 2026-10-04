---
icon: blocks
---

[English](../en/content-registration.md)

# 内容注册

包：`com.huidu.farmersdelight.api.registry`

`ContentRegistration` 让附属插件用自己的命名空间注册 CraftEngine 的**内容类型** —— 方块行为、物品行为、通用 function、通用 condition 与战利品 function —— 这样它自己的 CraftEngine 包就能像 FarmersDelight 的包一样，在 YAML 里引用这些类型。

本页涉及的类型：

| 类型                         | 形态        | `@ApiStatus`     |
| -------------------------- | --------- | ---------------- |
| `ContentRegistration`      | 静态门面，私有构造 | `@NonExtendable` |
| `ContentRegistration.Kind` | 枚举，5 个常量  | —                |

这个类没有 `@ApiStatus.Experimental`：五个注册方法、`isRegistered`、`registeredIds` 与 `unregister` 是已发布的接口面，其余成员全部是 `@ApiStatus.Internal`（见最后几节）。

## 能注册什么

```java
public static void registerBlockBehavior(Key id, BlockBehaviorFactory<?> factory);
public static void registerItemBehavior(Key id, ItemBehaviorFactory<?> factory);
public static <T extends Function<Context>>  void registerFunction(Key id, FunctionFactory<Context, T> factory);
public static <T extends Condition<Context>> void registerCondition(Key id, ConditionFactory<Context, T> factory);
public static void registerLootFunction(Key id, LootFunctionFactory<?> factory);
```

factory 参数都是 CraftEngine 自己的类型，所以实现它们就意味着要对着 CraftEngine 编译 —— 其它接收 CraftEngine `Key` 或包装类的工具方法也是如此。`Kind` 枚举给出五个注册表（`BLOCK_BEHAVIOR`、`ITEM_BEHAVIOR`、`FUNCTION`、`CONDITION`、`LOOT_FUNCTION`），`Kind.label()` 返回 FarmersDelight 在消息与晚注册告警里用的可读标签。

类型落进的注册表就是 FarmersDelight 自己内容用的那一张，因此包文件可以在 CraftEngine 接受该类条目的任何位置引用你的类型，不必考虑注册先后。

## id、命名空间与冲突

id 必须**带命名空间** —— `myplugin:my_type`，而不是裸 `my_type`：

* 命名空间或值为空会抛 `IllegalArgumentException`；
* `minecraft` 与 `farmersdelight` 两个命名空间**被保留**，同样抛 `IllegalArgumentException`。能覆盖 FarmersDelight 自身类型 id 的包会改变服务器上每一个包的行为，所以这门面拒绝成为那条路 —— 请在自己的插件命名空间下注册；
* `null` 的 id 或 factory 抛 `NullPointerException`。

重复注册会被**拒绝**，绝不静默替换。分两种情况，抛出的消息不同：

* 同一个 kind 下由你重复注册同一个 id —— `IllegalStateException`，消息点名该 kind；
* CraftEngine 已经知道这个 id —— 无论是 FarmersDelight 自己的注册还是别的插件的 —— `IllegalStateException`，消息为「already registered with CraftEngine」。这种情况下你的 factory 完全不会被应用，已有类型保持原行为。

各 kind 是**各自独立的注册表**，所以同一个 id 可以同时是一个 function 和一个 condition；只有同一张表内的重复才算冲突。`registeredIds()` 反映了这一点：返回当前通过这个类注册的全部 id，跨 kind 去重，顺序不保证。

## 查询与撤销

```java
public static boolean isRegistered(Key id);      // 该 id 在任意 kind 下是否已注册
public static Set<Key> registeredIds();          // 全部 id，去重、不可修改
public static boolean unregister(Key id);        // 返回是否真的移除了东西
```

`unregister` 也接受 `null` 并返回 `false`。它有一条来自 CraftEngine 而非 FarmersDelight 的硬限制：

**CraftEngine 没有注销已注册类型的 API。** `unregister` 只是让这个门面不再重放该条目并把它忘掉，但类型本身会在 CraftEngine 里保留到本次 JVM 运行结束。已经在用该 id 的内容继续可用；而只要 CraftEngine 还持有这个类型（本次运行内一直如此），这个 id 就**不能再注册** —— 替换用的 factory 永远不会被查询，之后再注册同一 id 会抛 `IllegalStateException`，消息为「unregistered earlier」那一类。需要换实现时请用**新的 id**，不要撤销后重注册。

## 晚注册

在 CraftEngine 加载完内容之后再注册仍然会成功：条目会被应用，但**已经存在的内容要等 CraftEngine 重读数据包之后才会用上它** —— 也就是 `/ce reload`（或重启）。这种情况会通过控制台键 `plugin.content_registration_late` 报出 id 与 kind，让服主知道某个类型为什么像是被忽略了，而不是去猜。

尽量在附属能最早的时刻注册以避免这条告警：一旦 CraftEngine 报告内容已加载，告警就是唯一的反馈，而且类型要等到下一次 `/ce reload` 才会到达已有内容。若你的附属同时响应 CraftEngine 自己的重载事件，重复注册同一个 id 就是冲突 —— 用 `isRegistered(id)` 守卫，或只注册一次。

## 重放是防御性的，不是修复步骤

FarmersDelight 自己的注册 pass 会按 kind 调用 `ContentRegistration.apply(Kind)`。该 pass 只重放 CraftEngine 不再报告为已注册的条目，而且是幂等的 —— 对一张完好的注册表再跑一遍什么也不会应用。

本插件针对的 CraftEngine 版本在整个 classloader 生命周期内都保留内置类型注册表，所以**CraftEngine 重载本来就不会丢掉外部注册**：重放是纵深防御，不是重载所依赖的步骤。不要围绕「某个 CraftEngine 构建会按重载重建注册表」来设计，也不要把重放理解成「CraftEngine 真的丢弃过的类型一定能恢复」的承诺。

## 内部成员与调用位置

`Bridge`、`apply(Kind)`、`installBridge(Bridge)` 与 `resetForTests()` 都是 `@ApiStatus.Internal`。它们存在的意义是让这套记账与校验能在没有运行 CraftEngine 的情况下被测试；生产代码从不调用它们，附属也不应调用。

和其它 api 类一样，请**在方法体内**调用这些方法 —— `onEnable`、监听器、命令处理器。不要在附属的字段初始化、静态块或 `static final` 常量里引用 `ContentRegistration`：那会让这个类在附属的类初始化期被加载，而附属跑在自己的类加载器下，一旦那里失败（`NoClassDefFoundError`），初始化会被中止，你正在装配的东西会被静默禁用。`CompatAttributes` 上的同一条规则见[版本兼容工具](compat-utilities.md)。

## 可用性

没有 `hasFeature` id 覆盖这个类，因此也没有对应的探测项。它属于提供它的那些构建所发布的 `api.**` 接口面，旧构建根本缺少这个类 —— 任何引用到它的附属路径都会以 `NoClassDefFoundError` 失败。要么要求使用带它的构建，要么只把引用放在确认类存在之后才会走到的方法体里。

## 示例

```java
import com.huidu.farmersdelight.api.registry.ContentRegistration;
import net.momirealms.craftengine.core.util.Key;

// 在 onEnable() 内，也就是方法体内
ContentRegistration.registerBlockBehavior(Key.of("myaddon:custom_keg"), MyKegBehaviorFactory.INSTANCE);
ContentRegistration.registerLootFunction(Key.of("myaddon:extra_drop"), MyExtraDropFunction.FACTORY);
```

之后包文件里就能像使用 `farmersdelight:*` 类型那样，把 `myaddon:custom_keg` 用作方块行为类型、把 `myaddon:extra_drop` 用作战利品 function。

## 相关页面

* [快速上手](getting-started.md)
* [方块与工作站](blocks-and-stations.md)
* [配方包总览](recipes-overview.md)
* [版本兼容工具](compat-utilities.md)
