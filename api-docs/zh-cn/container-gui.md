---
icon: layout-grid
---

[English](../en/container-gui.md)

# 容器 GUI

包：`com.huidu.farmersdelight.api.gui`

这个包是容器 GUI 背后的共享引擎：查看者一侧是字符网格**布局**，工作站一侧是扁平**存储**。配方书是同一套形状 —— [RecipeBookLayout](recipe-book-layout.md) 继承 `GuiLayout` —— 所以一个格子在两边含义一致。

本页涉及的类型：

| 类型                                       | 形态            | `@ApiStatus`      |
| ---------------------------------------- | ------------- | ----------------- |
| `GuiLayout`                              | 接口，只读视图       | `@OverrideOnly`   |
| `GuiLayouts`                             | 静态门面，私有构造     | —                 |
| `GuiSlotGroup`                           | record        | —                 |
| `GuiWriteBack`                           | 静态门面，私有构造     | —                 |
| `ContainerGuiKernel`                     | final，引擎      | —                 |
| `ContainerGuiKernel.Controller`          | 工作站实现的接口      | —                 |
| `GuiItems`                               | 静态门面，装饰物品构建器  | —                 |

## GuiLayout：可寻址不等于可写回

```java
@ApiStatus.OverrideOnly
public interface GuiLayout {
    String BACKGROUND = "background";

    int rows();
    default int size();                                  // rows() * 9
    default boolean contains(int rawSlot);               // 0 <= rawSlot < size()
    boolean isFunctional(int slot);
    default int[] functionalSlots();                     // 升序，新数组
    int[] slotsOf(String type);                          // 升序，新数组；未知或 null 时为空数组
    default int firstSlotOf(String type);                // 没有则 -1
}
```

legend 类型为 `background` 的格子 —— 以及网格之外的格子 —— 是装饰；其余格子都是**功能格**，也就是属于布局内容、可以被寻址的格子。

**功能格不等于可写回。** 工作站上那些只用于显示的格子（进度条、输出槽、成品预览）在这里是功能格，但 GUI 仍然绝不能把查看者的值写回去。`isFunctional` 回答的是「这个格子装内容吗」，不是「查看者能改它吗」；写回判定属于 `GuiWriteBack` / `Controller.mayCommit`。

之所以把 `isFunctional` 作为原始方法，是因为各实现的内部结构不同 —— 工作站读取器从图标表回答，解析出来的视图扫描预先算好的格子数组。`functionalSlots`、`slotsOf`、`firstSlotOf` 都是派生的，并且每次返回**新数组**，所以调用方可以留存、排序或修改拿到的东西而不影响布局。热路径应当在每 tick 之外调用一次 `functionalSlots()` 并缓存。传入未知或 `null` 类型返回空数组，绝不返回 `null`。

## GuiLayouts：通用读取器

`GuiLayouts` 是 `gui.yml` 布局读取器的通用那一半，不碰 CraftEngine、不碰服务端：它只读 `rows`、`layout` 与 `legend`，诊断也只是纯数据变换。

```java
public static GuiLayout parse(ConfigurationSection section);       // section 为 null 时返回 null
public static String[] cellTypes(int rows, List<String> layout, Map<Character, String> legend);
public static int warnUnknownCharacters(Logger logger, String configPath, int rows,
                                        List<String> layout, Map<Character, String> legend);
```

`parse` 读取 rows（默认 3，至少 1）、`layout` 与 `legend`（单字符键），返回的视图**只**回答 `GuiLayout` —— 标题、物品以及工作站专属的槽位角色仍留在调用方自己的配置类里。

`cellTypes` 返回归一化后的网格：`rows * 9` 项、按槽位顺序、永不为 `null`，而且每次调用都是**新数组**。任何「不是画出来的、且 legend 有可用条目的格子」都读作 `background`：

* 缺失的行、不足九个字符的行，或值为 `null` 的行；
* 空白字符；
* legend 没有定义的字符；
* legend 条目为 `null` 或空白。

超出 `rows` 的行与超出第九列的字符会被忽略；rows 小于 1 得到空数组。

### 与 `GuiConfig.getSlotType` 的一处不一致

通过自己的配置类读同一张网格的工作站还有一份字面查询。两者是**有意**不同的：

* `GuiConfig.getSlotType(slot)` 对网格没有画出的格子（缺失的行，或超出该行长度的列）返回 `null`，并且**原样**返回 legend 的值 —— 所以写得没有值的 legend 键会保持 `null`；
* `GuiLayouts.cellTypes` / `parse` 会**归一化**：同样的格子与条目都变成 `background`。

两者在「画出来的、legend 条目为非空白类型」的每个格子上都一致；只在 legend 把画出的字符映射到 `null` 或空白字符串的地方不同。也正因如此，配方书自己的原始方法 `RecipeBookLayout.slotType(int)` 采用与 `cellTypes` 相同的归一化，三者随后对每个画出的格子口径一致。

### warnUnknownCharacters

记录三类网格问题并返回记录了几条：

* 行数与 `rows` 不一致；
* 某一行不是九个字符宽；
* 画出的字符 legend 没有定义。

它刻意做成自包含的：接收 logger 与配置路径，而不是去抓插件，所以附属可以用自己的 logger 上报，不涉及静态查找也不涉及语言键。`null` 的 logger 或 layout 什么都不记，空白字符不是问题，路径为空白或 `null` 时报作 `gui.yml`。

## GuiSlotGroup：一个类型对应一段连续存储

```java
public record GuiSlotGroup(String type, int storeStart, int count, @Nullable String iconType) {
    public int storeIndex(int offset);      // storeStart + offset
}
```

容器 GUI 一侧是配置出来的格子，另一侧是扁平的存储值数组；这个 record 就是两者的衔接。`type` 选出格子（`layout.slotsOf(type)`），`storeStart` 是第一个格子对应的存储下标，`count` 限制其中有多少个承载值 —— 布局画出的同类型格子多于存储容量的部分，就是纯装饰格。

`iconType` 指定「存储值为空时画进该格的占位物」，当空格子直接清空时为 `null`。物品本身来自 controller（`Controller.icon(iconType)`），因为只有工作站知道它的占位物从哪来。

## GuiWriteBack：这份快照还能提交吗？

每个容器 GUI 都给查看者一份自己的快照库存，而原版是在一个 tick 之后才应用点击，所以要写回的值可能是在工作站 ticker 跑之前画上去的。盲目提交会把工作站已经花掉的物品还回来，或者把期间收到的物品抹掉 —— **物品不是变成两份，就是凭空消失**。

```java
public static boolean mayCommit(@Nullable ItemStack stored, @Nullable ItemStack paintedBaseline);
public static boolean mayCommit(boolean storedEmpty, boolean paintedEmpty, boolean similar,
                                int storedAmount, int paintedAmount);
```

这条规则是写回策略的**唯一来源**：

* `null` 与空气是同一件事（空格子）；
* **两侧都空** → 允许提交；
* 否则**同物品、同数量**（`isSimilar` 且数量相等）→ 允许提交；
* 其余情况 → **拒绝**。

**拒绝不是「保留查看者手里那份」。** 返回 `false` 意味着调用方必须把这个格子按存储重绘；静默保留 GUI 里的值，正是这道守卫要防的复制或丢失。内核已经替你完成了这次重绘。

比较是纯数据的：不读世界、不克隆、任何线程都能跑。只接收值的那个重载是为了让这条规则在没有运行服务端时也能被断言。

## ContainerGuiKernel

```java
public ContainerGuiKernel(Controller controller, GuiSlotGroup... groups);
public boolean hasLayout();
public boolean fill(Inventory view);
public boolean refreshAll(Inventory view);
public boolean refreshAll(Inventory view, IntPredicate skipPending);
public boolean refreshSlot(Inventory view, int rawSlot);
public boolean syncAll(Inventory view);
public boolean syncSlot(Inventory view, int rawSlot);
public int storeIndex(int rawSlot);                       // 该槽不装内容时返回 -1
```

内核负责每个容器 GUI 都要重复的那部分：把配置格子映射到存储下标、记住每个格子是从什么画出来的、拒绝过期写回并把被拒的格子按存储重绘、以及「编辑仍在途中」的跳过规则。

* `fill` 先给布局不使用的每个格子画背景，再画各组存储值（空格子画占位物）。
* `refreshAll` 把所有配置格子按存储重绘；`refreshAll(view, skipPending)` 跳过 `skipPending` 接受的格子 —— 编辑仍在途中的格子拥有玩家即将取走的那份值，按存储重绘会让物品显示两次并被发放两次。
* `refreshSlot` 重绘一个配置格子，空格子恢复占位物；不是配置格子的槽位保持不动。
* `syncAll` 提交守卫允许的每个配置格子，重绘它拒绝的格子，并在之后调用一次 `markDirty()`。
* `syncSlot` 是单格版本，用于原版延迟的点击与拖拽同步。守卫拒绝的格子按存储重绘；不是配置格子的槽位保持不动（并返回 `true`）。
* 工作站没有布局时，上述方法都返回 `false`，调用方可以退回硬编码网格；此时 `storeIndex` 返回 `-1`。

### Controller 契约

```java
public interface Controller {
    @Nullable GuiLayout layout();                 // null = 所有引擎方法都报告「什么都没做」
    int slotCount();
    @Nullable ItemStack stored(int storeIndex);   // 权威值；不得记录任何东西
    void store(int storeIndex, @Nullable ItemStack item);
    Object storeLock();                           // 真正的监视器对象，绝不为 null
    void markDirty();

    default @Nullable ItemStack icon(@Nullable String iconType);                 // null
    default @Nullable ItemStack backdrop();                                      // null
    default boolean isPlaceholder(ItemStack stack, @Nullable String iconType);   // false
    default boolean mayCommit(@Nullable ItemStack stored, @Nullable ItemStack paintedBaseline);
}
```

* `stored` 必须是纯读取。每次绘制内核都会调用它，提交时还会再调用一次；在那里记录东西（一个「上次所见」字段、一个脏标记）等于让绘制变成对存储的修改。
* `store` 负责克隆与归一化：它收到的可能是格子里原来的任何东西，包括占位物位置传来的 `null`，必须克隆自己保留的内容，并且存 `null` 而不是空 stack。引擎从不把活的 stack 交给存储。
* `storeLock()` **真的会被使用**：内核在每次访问存储以及读写每格绘制记录时都会获取它。它必须是真正的对象、绝不为 `null`，而且是可重入的 —— 常见调用形态（点击处理器把整段读-改-写包在同一把锁里）不受影响。
* `markDirty()` 在提交改变了存储时被调用，好让工作站把方块实体标脏。内核每次 `syncAll` 调用一次，`syncSlot` 只在真正提交时调用。
* `mayCommit` 默认走 `GuiWriteBack.mayCommit`。只给提交策略确实不同的工作站（例如带符号的增量）才覆盖它 —— **更弱的覆盖正是一个容器开始复制物品的方式。**

## 线程与区域契约

这是 Folia 上不能跳过的一段。

* `fill`、`refreshAll`、`refreshSlot` 只写**查看者**的库存，因此必须在拥有该查看者的区域上运行。
* `syncAll` 与 `syncSlot` 读存储并写回。它们在 **`Controller.storeLock()`** 之下运行 —— 内核在每次访问存储以及读写绘制记录时都会获取它，所以在查看者区域上的调用方不可能与存储区域上的 ticker 交错。
* **引擎自身不派发。** 它从不调用调度器、从不读世界、从不把工作跨区域搬运。如果查看者与存储分属不同区域，安排这件事是调用方的责任：用 api 的调度辅助（`FarmersDelightApi.runAtLocation` / `runLaterAtLocation`，见[调度](scheduling.md)）分别调度两侧。在 Folia 上「查看者区域 ≠ 存储区域」是常态，不是边角情况。

忽略这一点的容器，要么从错误的线程写查看者的库存，要么让存储所在区域阻塞在一次跨区域读取上 —— 这正是上面这条拆分属于已发布契约、而不是实现细节的原因。

## 使用它的工作站长什么样

```java
GuiLayout layout = GuiLayouts.parse(section);            // rows / layout / legend
if (layout != null) {
    ContainerGuiKernel kernel = new ContainerGuiKernel(controller,
            new GuiSlotGroup("bait", 0, 1, "bait_icon"),
            new GuiSlotGroup("catch", 1, 3, null));

    kernel.fill(view);                                   // 查看者所在区域
    // ... 之后，在存储所在区域、或在 storeLock() 之内：
    kernel.syncSlot(view, rawSlot);
}
```

`GuiItems.build(section)` / `GuiItems.build(section, placeholders)` 按 FarmersDelight 自己 GUI 使用的同一套 `items:` 段形状构建装饰物品（物品 id、material、custom-model-data、item-model、name / lore 走 MiniMessage 与字形标签，以及可翻译的 `name-key` / `lore-keys` 对），所以附属的装饰也能像家族里其它文本一样本地化。

## 相关页面

* [RecipeBookLayout 与打开配方书](recipe-book-layout.md)
* [方块与工作站](blocks-and-stations.md)
* [调度与 ApiTask](scheduling.md)
* [版本兼容工具](compat-utilities.md)
