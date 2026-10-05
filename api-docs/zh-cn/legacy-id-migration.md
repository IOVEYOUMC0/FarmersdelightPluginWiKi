---
icon: right-left
---

[English](../en/legacy-id-migration.md)

# 旧 id 迁移

包：`com.huidu.farmersdelight.api.migration`
类：`LegacyIdMigration` —— `final`，私有构造，`@ApiStatus.NonExtendable`。

`LegacyIdMigration` 让插件声明"过去存在、现在有了替代品"的物品 id，于是世界里已经存在的那些堆叠会在
FarmersDelight 下次接触到它们时被改写。它是**静态门面**，没有 `getInstance()`。

## 为什么需要它

CraftEngine 没有针对已注册 id 的别名或改名机制。改掉一个 id，旧 id 就直接不存在了：所有仍带着它的堆叠
都会变成未知物品 —— 玩家物品栏、箱子、潜影盒、末影箱、展示框里的都一样。

上游 Farmer's Delight 模组用 NeoForge 的 `DeferredRegister.addAlias` 解决同一个问题 —— 它 1.4 版把
`barbecue_stick` 改名为 `cooked_meat_skewer` 而没破坏老存档，靠的就是这个。CraftEngine 没有对应能力，
所以 FarmersDelight 自己开了一层：声明 `旧 id → 替代品`，插件负责改写它能接触到的堆叠。附属用同样的
方式声明自己的 id。

上游那份别名表里有两个条目正好说明这个机制要解决的问题：

* `barbecue_stick → cooked_meat_skewer` —— 一次物品改名；
* `basket → bamboo_basket` —— 上游对**方块和物品**都做了别名，而 FarmersDelight 的包里至今仍在发布
  `farmersdelight:basket`，这正是这一层存在的场景。注意本 api 只迁移**物品**：方块层面的 id 不在范围内
  （见*边界*）。

## 接口

```java
public static void registerItem(Key legacyId, ItemStack replacement);
public static void registerItem(Key legacyId, Key currentId);
public static boolean isLegacy(ItemStack stack);
public static ItemStack migrate(ItemStack stack);

// 只读辅助
public static String resolveId(String legacyId);   // 最终解析到的 id，未注册时为 null
public static boolean isEmpty();                   // 还没有任何注册
public static int size();                          // 已注册的旧 id 数量
public static int conflictCount();                 // 因重复而被拒绝的注册数
```

`Key` 是 CraftEngine 的 `net.momirealms.craftengine.core.util.Key`，与 `ContentRegistration` 用的是同一个类型。

### registerItem(Key legacyId, ItemStack replacement)

最常用的形式。声明带着 `legacyId` 的堆叠变成 `replacement`，而这个 `replacement` 的 meta 同时就是迁移
结果的**模板**：需要自定义名、lore、附魔或耐久，就把想要的那份物品堆叠传进来。

以下情况会被静默忽略（不报错也不生效）：`legacyId` 为 `null`、`replacement` 为 `null` 或空气、或者
`replacement` 的 CraftEngine id 解析不出来。模板按"替代品解析到的 id"存放，因此链式映射共用同一份模板。

### registerItem(Key legacyId, Key currentId)

便捷重载：迁移结果由 `currentId` 自身的物品定义创建，因此不带自定义 meta。数量始终取自旧堆叠。

### isLegacy(ItemStack stack)

当堆叠携带的 id 已被登记为旧 id 时返回 true。`null`、空堆叠、以及"还没有任何注册"时一律返回 false。

### migrate(ItemStack stack)

返回旧堆叠的替代品，或原样返回。契约如下：

* `null` 进 `null` 出；空堆叠原样返回。
* 未注册的 id（包括已经迁移过的堆叠）返回**同一个实例**。请用身份判断（`migrated == original`）来确定
  是否发生了变化 —— 插件自己的钩子就是这么做的。
* 已注册的 id 变成替代品，**数量保留**。
* **持久化数据是合并**：替代品自带的条目优先 —— 其中包含 CraftEngine 自己的 id 键，所以结果确实就是新
  物品 —— 只有旧堆叠才有的条目会被带过去。
* **不搬运**：自定义名、lore、附魔、耐久。需要这些就改用 `registerItem(Key, ItemStack)` 传一份准备好
  的替代品。
* **幂等**：新 id 不在旧 id 表里，因此再调用一次不会产生任何变化。
* 替代品没有可用定义时，原样返回旧堆叠，而不是把它删掉。
* 链式映射一次解析到底 —— `A → B → C` 直接得到 `C` —— 上限 8 跳，因此成环也能终止。
* 表为空时立即返回，这正是"附属没注册之前自动钩子零开销"的原因。
* **不做任何世界访问**。它只读传入的那个堆叠，因此任何线程、任何区域都可以调用。自动钩子则总是在物品栏、
  方块或实体的所有者线程上调用它。

## 前提：旧定义必须留在包里

**旧 id 必须在产出它的那个包里仍然有物品定义。** CraftEngine 是拿 id 去对照已加载的定义解析的；一旦旧
定义没了，世界里已有的堆叠会先被变成**未知物品**，这一层根本看不到可用的 id，`isLegacy` 永远不会匹配。
旧 id 已经没有定义的迁移表就是死代码。

所以一次改名的正确顺序是：

1. **旧定义继续留在包里** —— 即使它现在指向和新物品相同的模型与贴图。与新 id 同一个版本上线。
2. 在 `onEnable`（或你注册内容的地方）登记映射，同样在同一个版本里完成。
3. 之后才停止**发放**旧物品：删掉产出或引用它的配方 / 标签 / 语言条目，让新的旧 id 堆叠不再产生。定义
   本身可以一直留着 —— 没人引用的定义很便宜，而提前删掉它正是让迁移失效的原因。

## 自动钩子

五个钩子会在堆叠进入视野时改写它们，你不需要自己调度。

| 时机 | 改写什么 | 运行在 |
| --- | --- | --- |
| 玩家进服 | 玩家物品栏与末影箱 | 玩家所在线程 |
| 打开容器 | 被打开的容器，每次打开一次（不是每次点击） | 拥有该方块的区域 |
| 我们自己的容器 GUI | GUI 顶层物品栏里的镜像，在其载入边界 | 查看中的玩家线程 |
| 掉落物生成 | 刚掉出的那份堆叠 | 掉落物实体所在线程 |
| 启动扫描 | 已加载区块里方块实体容器中的物品 | 每个已加载区块一次派发 |

所有钩子走同一条判定：当前线程**已经拥有**目标时内联执行，否则交给所有者区域，并且**恰好派发一次**。
任何钩子都不会读取不属于自己的容器，也都**不使用 `Bukkit.getScheduler`**。单线程 Paper 服务端上这些都
是内联工作；Folia 上跨区域的情况才会多一次调度任务。启动扫描只走一遍已加载区块，且在每个区块自己的区域
内重新确认一次，因此已卸载的区块会被跳过。

### 哪些容器会被处理

只有当容器的 **owner 能被指名** 时才会改写：

* `BlockState` 持有者 —— 该方块的位置；
* `DoubleChest` 的**任意一半** —— 优先取左侧，仅为了结果确定；
* FarmersDelight 自己的容器 GUI —— 归查看它的玩家。

其他任何持有者 —— 第三方持有者插件、商人、本插件无法归属的自定义 `InventoryHolder` —— 一律**跳过**，
且这次打开不会被记为"已迁移"，因此下次打开还会重试。跳过的情形是：在没有弄清哪个区域拥有它的情况下，
从碰巧看到该事件的线程去读一个容器，正是这些钩子绝不能做的事。所以被跳过的容器要么迁移得更晚（后续某次
打开时），要么在再也不会被打开的情况下永远不迁移。

### 钩子覆盖不到的地方

钩子只覆盖 FarmersDelight 自己会接触到的存储：玩家物品栏与末影箱、被打开的容器、本插件容器 GUI 的载入
边界、世界里生成的掉落物，以及启动时已加载区块里的方块实体容器。**附属自己的存储不在这个列表里** ——
自定义 GUI 缓存、数据库列、配方书缓存、或者你自己 YAML 里的物品引用。

这些必须在你自己的边界上显式调用：

```java
// 自有存储里的堆叠：没有匹配时 migrate() 返回同一个实例
ItemStack fixed = LegacyIdMigration.migrate(storedStack);
if (fixed != storedStack) {
    saveToMyStorage(fixed);
}

// 存的是 id 字符串时（配置文件、数据库列里的 "myaddon:old_widget"）
String current = LegacyIdMigration.resolveId(storedId);   // 不是旧 id 时为 null
if (current != null) {
    saveToMyStorage(current);
}
```

## 配置开关

`config.yml`：

| 开关 | 作用 |
| --- | --- |
| `legacy-id-migration.enabled` | `false` 时五个自动钩子全部停用 |
| `legacy-id-migration.log-summary` | `true` 时每次运行写一行汇总 |

`log-summary` 写的是 `Legacy id migration: rewrote N stack(s) across M inventory(ies).`，而且只在总数比
上一行增长时才写，所以繁忙的服务器不会变成"每个容器一行"。

**把 `enabled` 关掉并不会停用接口。** `LegacyIdMigration.migrate(stack)` 两种情况都始终可用 —— 这正是
附属应该在自己的存储边界上主动迁移、而不是只依赖自动钩子的原因。钩子只是为 FarmersDelight 本来就会接触
到的那些存储提供的便利。

**把它再打开并不会补跑启动扫描。** `enabled` 会被 `/fd reload` 重新读取，但这条路径只重读两个开关值；
扫描只在启动时跑一次，不会重跑。所以 `false → true` 之后，已经加载区块里的堆叠要等到下一次自然接触 ——
玩家进服、或有人打开那个容器。要在运行中的服务器上强制全量跑一遍，就重启：启动是唯一的自动扫描时机。

## 示例

```java
import com.huidu.farmersdelight.api.migration.LegacyIdMigration;
import net.momirealms.craftengine.core.util.Key;

// 写在 onEnable() 里，也就是方法体内
LegacyIdMigration.registerItem(
        Key.of("myaddon:old_widget"),   // 老存档里可能存在的旧 id
        Key.of("myaddon:new_widget"));  // 它要变成哪个物品

// 替代品需要自己的名字、lore 或附魔时，传一份准备好的堆叠：
LegacyIdMigration.registerItem(Key.of("myaddon:old_gem"), preparedNewGem());
```

在你自己的存储边界上：

```java
ItemStack stored = readFromMyCache();
ItemStack fixed = LegacyIdMigration.migrate(stored);
if (fixed != stored) {
    writeBackToMyCache(fixed);   // 身份判断就是"变了没有"
}
```

这些调用要写在**方法体内** —— `onEnable`、监听器、命令处理器。不要在字段初始化器或 `static final` 常量
里引用 `LegacyIdMigration`：那会让你自己的类初始化阶段就去加载这个类，一旦失败
（`NoClassDefFoundError`）初始化器会中断，你正在接的东西会被静默地留成关闭状态。这条规则对每个 api 类都
适用 —— 见[版本兼容工具](compat-utilities.md)。

## id、链与冲突

id **区分大小写**，只做 trim（MMOItems 的 id 带大写类型段）。`null` 与空白 id 被忽略。

同一个旧 id 用不同目标注册两次时，**保留第一条**，并把第二条通过插件 logger 报告一次 —— 两个附属无法
悄悄争夺同一个 id。重复注册完全相同的映射则是空操作。被拒绝的注册数量可由 `conflictCount()` 读到。

## 边界

* **只处理物品。** 方块 id 的改名目前不在范围内。
* **不是永久别名。** 它是在 FarmersDelight（或你自己的调用）接触到堆叠时的一次性改写；从不被加载的堆叠
  永远不会被改写。它不是"让旧 id 永远能解析"的查询表。
* **不管数据包文件。** 引用旧 id 的配方、标签、进度与语言键都是文件而不是堆叠 —— 要各自单独更新。
* **无法复活已删除的定义。** 删掉旧物品定义之后，旧堆叠在迁移能帮忙之前就已经是未知物品；这正是上面那条
  前提重要的原因。
* **覆盖不到附属自己的存储。** 堆叠或 id 只存在于你的插件能读到的地方 —— 缓存、数据库、自定义 GUI、你自己的
  YAML —— 就要在那里自己调用 `migrate(ItemStack)` 或 `resolveId(String)`；钩子看不到它们。
* **重新打开开关不会补扫。** 见*配置开关*：对已加载区块的自动扫描只在启动时跑一次，因此 `false → true`
  之后那些堆叠要等到被接触时才会迁移。
* **数量与持久化数据的落地只能实机验证。** 离线测试锁住的是映射规则与一次迁移的调用序列，它无法真正写入并
  读回结果：`ItemStack` 的子类可以离线构造，但 `setAmount`、`getType`、`getItemMeta` 与持久化数据容器都由
  Craft 实现，没有服务端就会抛异常。请把"数量保留"与"数据已合并"当作需要在测试服上确认的行为。

## 可用性

没有 `hasFeature` id 覆盖这个类，因此没有探测手段。它属于发布它的那些构建的 `api.**` 公开面，更老的构建
里根本没有这个类 —— 任何引用路径都会以 `NoClassDefFoundError` 失败。要么要求一个带它的构建，要么把引用
放在"确认过类存在之后才会走到的方法体"里。

## 相关页面

* [内容注册](content-registration.md)
* [物品](items.md)
* [版本兼容工具](compat-utilities.md)
