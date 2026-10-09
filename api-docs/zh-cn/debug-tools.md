---
icon: bug
---

[English](../en/debug-tools.md)

# 调试工具：DebugToolExtension 与 DebugToolRegistry

FarmersDelight 有一条管理员命令 `/fd debugtools`，用于压力测试：批量放置方块、批量激活让它们真的开始 tick、检查实时状态、校验已加载配方并撤销放置。`com.huidu.farmersdelight.api.util` 里的这两个类型让附属把自己的方块挂进这条命令，而不必自己再做一套调试 CLI。

对应的 feature id 是 `debug-tools`。

按功能测量耗时使用 `/fd stats profile 200 all`，也可把 `all` 换成 `handheld`、`handheld_display`、`cooking_pot`、`skillet` 或 `stove`。普通版也支持此命令。参数、计时范围及结果含义见[性能排查](../../server-guide/zh-cn/troubleshooting.md)。debug 构建输出带 `-debug.jar` 后缀，只与普通包二选一安装；`/fd perf` 始终指向 `/fd stats`。

debug 构建可以在小范围手动制造测试负载：

```text
/fd debugtools test cooking_pot 64 200
/fd debugtools test skillet 64 200
/fd debugtools test stove 64 200
/fd debugtools test all 128 600
```

命令会在玩家附近分批放置测试工作站、填入测试内容、启动对应功能采样；每轮最多处理 16 个位置，避免命令本身长时间占用服务器线程。`count` 是测试位置数，`ticks` 是采样时长（20-12000）。完成后使用 `/fd debugtools undo` 清理最近一批，使用 `/fd debugtools stop` 停止尚未完成的放置、激活或手持测试。`all` 只包含厨锅、放置煎锅和炉灶。

手持计算负载使用 `/fd debugtools test handheld 200 200`：按数量模拟独立会话，无需手持物品、持续右键或附近热源，不消耗玩家物品。输出更新、显示构建、完成次数和每模拟 tick 计算耗时；仅覆盖槽位匹配、进度、显示副本及配方完成核对，不包含网络、经验事件、真实背包或 Folia 多区域派发，不能当作 200 位真实玩家的 MSPT。模拟使用已加载的生牛肉营火配方原始时间；`stop` 输出当前结果并结束，退出时清理。真实手持链路继续用 `/fd stats profile 200 handheld` 和 `handheld_display` 采样。
推荐层级指令：

```text
/fd debugtools place recipe cooking_pot farmersdelight:beef_stew 4
/fd debugtools place random all 16
```

旧 `placeRecipe`、`placeRecipeRandom` 别名保留。undo 只处理自己的放置批次；独立 activate 没有独立撤销，已领取/消耗的产物不会恢复。stop 停止待执行批次、模拟任务和当前测试采样，不是完整回滚。


`recipe validate` 会检查已加载的厨锅和砧板配方是否为空、是否缺少成品/容器，以及标签是否解析不到任何原版物品、CraftEngine 物品或已注册的公共标签成员。解析阶段失败的配方仍会由常规 `recipe.load_failed` 日志报告。

## 保存和序列化采样

```text
/fd debugtools profile serialization 600
/fd debugtools stop
/fd stats
```

需要 debug 构建和 `farmersdelight.admin`；普通构建可用 `/fd stats profile 600 serialization`。debug 入口默认 600 tick，允许 20-12000 tick。采样器全服只运行一个会话，记录各区域真实调用，不按发起玩家筛选。

四类结果是实时容器保存、物品 NBT 编码、已接入保存锁的获取耗时和关闭快照的 holder 查询；输出调用数、平均、P95、最大和累计耗时。保存入口覆盖 FD、BAC、CAB、BBQD；停放标签直接写回不进入实时保存计时。物品编码也可能发生在打包等其它已接入路径，锁获取不覆盖外层早已持有的锁。

正常操作并等待自动保存即可；打开 GUI 本身不保证触发持久化。窗口内没有执行的路径显示零调用，需要时延长采样。命令不主动保存、卸载或重载区块，不改库存。停止后仍可用 `/fd stats` 查看结果。各行会嵌套，不能相加为 MSPT，也不包含 CE 整个区块编码、压缩及磁盘写入；配合 spark 定位调用栈和 GC。

## 前提：只有 debug 构建才会真的跑

`/fd debugtools` 只在 FarmersDelight 的 debug 构建中存在；在正常的发布构建上，命令类根本不在 jar 里，该命令也不会注册，所以请在 debug 构建上测试你的集成。在发布构建上，注册表大部分处于**休眠**状态：`register` 照常成功， 你的 `place`、`activate`、`cleanupBeforeUndo` 永远不会被调用。`status` 是例外——主插件常驻的 `/fd stats addon <name>`（见下文 `status` 小节）在普通构建里同样会调用它。

这就是为什么无条件注册是安全的，也是 FDAddonTemplate 和 BrewinAndChewin 都在 `onEnable` 里直接注册、不做 任何探测的原因。

## `DebugToolRegistry`

```java
package com.huidu.farmersdelight.api.util;

@ApiStatus.NonExtendable
public final class DebugToolRegistry {

    public static void register(DebugToolExtension extension);
    public static void unregister(String name);
    @Nullable public static DebugToolExtension find(String name);
    public static Collection<String> registeredNames();
    public static Collection<DebugToolExtension> all();
}
```

一个静态、进程级的 `ConcurrentHashMap`，键是小写后的扩展名。几点说明：

* `register` 幂等——同名重复注册会替换掉前一个。`extension` 为 `null`、或其 `name()` 为 `null` 时忽略。
* `unregister(name)` 必须在你的 `onDisable` 里调用。这张表是静态的，活得比你的插件类加载器久；留下陈旧 条目就等于让一个已经拆掉的 manager 一直可达，并在下一次 `/fd debugtools` 时被调用。
* `find` 对未知或 `null` 的名字返回 `null`。`registeredNames()` 与 `all()` 返回不可修改视图；命令的 Tab 补全用的就是 `registeredNames()`。

```java
@Override
public void onEnable() {
    // ...
    DebugToolRegistry.register(new ExampleDebugExtension(this));
}

@Override
public void onDisable() {
    DebugToolRegistry.unregister("example_block");
}
```

## `DebugToolExtension`

```java
package com.huidu.farmersdelight.api.util;

@ApiStatus.OverrideOnly
public interface DebugToolExtension {

    String name();

    int place(Player player, Location origin, int count, int spacing, int layers, UndoSink undo);

    @FunctionalInterface
    interface UndoSink {
        void capture(Location loc);
    }

    default int activateAt(Player player, Location location) { return 0; }

    default List<String> status(Player player) { return List.of(); }

    default void cleanupBeforeUndo(Location location) { }
}
```

`@ApiStatus.OverrideOnly` 与 `NonExtendable` 正好相反：你**应该**实现这个接口，但不应该自己去调用它的方法 ——唯一的调用方是 FarmersDelight。把实现当成回调面来写。

### `String name()`

小写的目标关键词，也就是管理员输入的那个词：

```
/fd debugtools place keg 64 2 3
```

它同时用于 Tab 补全，以及作为 `/fd stats addon <name>` 里的扩展名。`DebugToolRegistry` 注册时会把它转小写，所以直接返回小写形式 以免混淆。

### `int place(Player player, Location origin, int count, int spacing, int layers, UndoSink undo)`

当 `/fd debugtools place <name> [count] [spacing] [layers]` 中的 `<name>` 不是 FarmersDelight 的内置目标时 被调用。返回实际放进世界的方块数。

网格由命令自己铺好，每个备好的单元调用一次 `place(...)`，所以每次调用收到的是：

* `count`、`spacing`、`layers` 都是 `1`；
* `origin` 在该单元**下方**一格——请填充 `origin` 在 Y 轴上加一的那一格；
* 撤销批次已经开启，你之后每次 `undo.capture(...)` 都会并入这一批。

网格来自管理员输入的 `count` / `spacing` / `layers`，而备好的单元总数由 `max-place-count` 限制，所以一次命令
最多只会要你放这么多方块。该单元是空的、可编辑的，指向别处的 `undo.capture(...)` 会被拒绝。填充该单元并返回
实际放进世界的方块数即可，单元内部的排布方式由你决定。

在改动每个目标方块**之前**调用 `undo.capture(loc)`。`UndoSink` 是只有一个 `capture(Location)` 的函数式 接口；FarmersDelight 会把该坐标的方块状态快照进当前撤销批次，供 `/fd debugtools undo` 还原。最终没有改变 状态的快照，在撤销时会被静默跳过。

BrewinAndChewin 酒桶实现的精简版：

```java
@Override
public int place(Player player, Location origin, int count, int spacing, int layers, UndoSink undo) {
    if (player == null || origin == null || origin.getWorld() == null) return 0;
    BlockDefinition kegBlock = CraftEngineBlocks.byId(KEG_BLOCK);
    if (kegBlock == null) {
        player.sendMessage("§c[BAC] keg block not registered with CraftEngine.");
        return 0;
    }

    int grid = Math.max(1, (int) Math.ceil(Math.sqrt(count)));
    int total = count * Math.max(1, layers);
    int placed = 0;
    for (int i = 0; i < total; i++) {
        int layer = i / count;
        int layerIndex = i % count;
        Location loc = new Location(
                origin.getWorld(),
                origin.getBlockX() + (layerIndex % grid) * spacing,
                origin.getBlockY() + 1 + layer,
                origin.getBlockZ() + (layerIndex / grid) * spacing);
        Block target = loc.getBlock();
        if (target.getType() != Material.AIR
                && !BlockStateUtils.isReplaceable(BlockStateUtils.getBlockState(target))) {
            continue;
        }
        if (undo != null) undo.capture(loc);
        if (CraftEngineBlocks.place(loc, kegBlock.defaultState(), false)) {
            placed++;
        }
    }
    return placed;
}
```

注意那句 `undo != null` 的防御性判断——目前命令总是会传入 sink。

### `int activateAt(Player player, Location location)`

可选，默认返回 0。主插件只向该方法传入管理员最近一批测试放置的坐标，按 16 项 / 2ms 预算切片，先验证加载状态、区域归属、建造权限和操作后的方块身份。只修改该坐标，另行检查附属自己的使用权限；不要遍历全世界 manager。

旧 `activate(Player)` 保留二进制兼容并标记弃用，调试命令不再调用它。`activate all` 也只处理自己的最近批次。附属放置和激活在 Folia 上暂时拒绝，以免尚未支持跨区域的扩展绕过归属检查。

### `List<String> status(Player player)`

可选，默认返回空列表。由主插件**常驻**的子命令 `/fd stats addon <name>` 触发（`/fd stats` 总览会把每个已注册扩展显示为可点击的名字，点进去即查看对应状态行），因此在发布构建上同样会被调用。每条返回一行，通过把 `{line}` 占位填进翻译键 `command.stats_addon_line` 发送给玩家（`{name}` 为扩展名），自身不带前缀。

```java
@Override
public List<String> status(Player player) {
    return List.of("tracked example_blocks (debug-placed): " + placed.size());
}
```

发送时这一行会作为 `{line}` 替换进 `command.stats_addon_line` 模板，所以不要在行里放字面的 `{line}` 或 `{name}`——它们会被当作替换占位；想带前缀/颜色就写在这一行里，或改那个翻译键。

返回 `null` 是被容忍的（等同于"没有行"），但文档化的退出方式是返回空列表。

### `void cleanupBeforeUndo(Location location)`

可选，默认空实现。在 `/fd debugtools undo` 期间对**每一个被还原的坐标**调用，且发生在方块数据被还原 **之前**。用它释放属于本扩展的内存跟踪、清掉方块实体 NBT，免得被撤销的放置留下幽灵状态。

调用点有两个性质值得知道：

* 它会对**所有已注册扩展**调用，而不只是放下这个方块的那一个。所以坐标不属于你时，你的实现必须以很低的 代价直接返回。
* RuntimeException / LinkageError 会记录并中止当前撤销批次，保留尚未清理的记录供重试；不会在附属清理失败后继续删除方块。

```java
@Override
public void cleanupBeforeUndo(Location location) {
    if (location == null) return;
    Iterator<Location> it = placed.iterator();
    while (it.hasNext()) {
        Location loc = it.next();
        if (loc.getBlockX() == location.getBlockX()
                && loc.getBlockY() == location.getBlockY()
                && loc.getBlockZ() == location.getBlockZ()
                && loc.getWorld() != null && loc.getWorld().equals(location.getWorld())) {
            it.remove();
            return;
        }
    }
}
```

## 线程与撤销范围

内置工作在玩家所属调度器分片运行，只处理当前线程拥有的已加载坐标；远处或保护拒绝的坐标会跳过。Paper 扩展回调必须同步完成，不要从回调内安排延迟世界修改，否则它不会进入已捕获的计数、身份检查和撤销批次。

撤销历史每位玩家最多八批。undo 核对操作后的 ID / 方块数据及容器 controller 身份，跳过后来替换的对象。它清理测试造景，不回收已经领取的食物、物品或插件外部事件副作用；直接通过其它插件替换为完全相同、无 controller 的方块，无法仅凭状态判断身份。stop 停止后续批次、模拟任务和该测试的采样，不停止已经开始的正常生产。

## 相关页面

* [版本兼容工具](compat-utilities.md)
* [快速上手](getting-started.md)
* [FarmersDelightApi 入口](farmersdelight-api.md)
