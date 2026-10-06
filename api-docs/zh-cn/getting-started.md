---
icon: circle-play
---

[English](../en/getting-started.md)

# 快速上手

FarmersDelight 是基于 CraftEngine 的 Paper / Folia 插件。附属（addon）是**一个独立的 Bukkit 插件**，与 FarmersDelight 跑在同一个 JVM 里，硬依赖 FarmersDelight，并在进程内直接调用它的 Java API。这里没有任何 网络协议，也没有命令桥接——你编译时链接一个 jar，运行时直接调方法。

本页讲的是构建配置、清单文件（`paper-plugin.yml`）的写法、`onEnable` 里该做什么，以及每个附属都应当装上的生命周期保护。

## 只有 `com.huidu.farmersdelight.api.**` 是稳定接口

这个包以外的内容都是内部实现，可能在版本更新时变化且不提供兼容保证。`api` 包就是全部契约：
它的签名只用 Bukkit 类型、JDK 类型、Adventure 类型和其它 `api` 类型，不会把内部实现泄漏到参数或返回值中。

反过来说，编译期你只需要这一个 api jar。

## 获取 api jar

FarmersDelight 的构建里有一个 `apiJar` 任务，只打包 `com/huidu/farmersdelight/api/**`，不含任何内部实现， 也不是一个能运行的插件：

```bash
# 在 FarmersDelight 仓库里执行
./gradlew apiJar
# -> build/libs/farmersdelight-plugin-1.0.3-api.jar
```

同一个 jar 也作为该模块的主产物发布，坐标见下。也就是说无论走哪条通道，附属拿到的都是同一份产物。

## 依赖这个 api

这个 jar 不需要到处复制：按坐标声明依赖即可：

```kotlin
compileOnly("com.huidu.farmersdelight:farmersdelight-plugin:1.0.3")
```

作用域用 `compileOnly`：运行时由真正的 FarmersDelight 插件提供实现。把 api jar shade 进自己的插件，只会多出 一份永远不会被用到的死类。

这个坐标有两条解析通道，两条通道用的是同一个版本号，也就是你的附属对应的 FarmersDelight 版本——这里是 `1.0.3`。

### 本地 composite build

当本地存在 FarmersDelight 检出（通常是同级的 `../FarmersDelight` 目录）时，构建会把它作为
[composite build](https://docs.gradle.org/current/userguide/composite_builds.html) 包含进来，该坐标会被替换成
那个检出的 `:apiJar` 产物。开发时应当走这条通道：离线可用，api 一改立刻生效，不需要发布任何东西。

```kotlin
// settings.gradle.kts
rootProject.name = "fdaddontemplate"

val farmersDelightCheckout = file("../FarmersDelight")
if (farmersDelightCheckout.isDirectory) {
    includeBuild(farmersDelightCheckout) {
        name = "farmersdelight-plugin"
    }
}
```

### git 源码依赖

没有这个检出时，同一个坐标改由 Gradle 的
[源码依赖（source dependency）](https://docs.gradle.org/current/userguide/declaring_repositories.html#sec:declaring_source_dependencies)
解析：Gradle 克隆 FarmersDelight 仓库，检出与所请求版本对应的 tag，构建它的 `apiJar`，再从那里解析产物。
整条链路不经过任何 Maven 仓库，两条通道编译时用的都是同一份只含 api 的 jar。

```kotlin
// settings.gradle.kts
sourceControl {
    gitRepository(uri("https://github.com/IOVEYOUMC0/Farmersdelight-Plugin.git")) {
        producesModule("com.huidu.farmersdelight:farmersdelight-plugin")
    }
}
```

这条通道需要网络，并且每个版本第一次解析时会先克隆一次，所以第一次 `build` 会明显比 composite build 慢。
只有 FarmersDelight 仓库打了对应 tag 的版本才能这样请求，因此请让版本号跟你要对标的发行版保持一致。

### 两条通道写在同一个脚本里

常见的附属 `settings.gradle.kts` 会同时声明两条通道，并在本地有检出时优先用检出：

```kotlin
val farmersDelightApiVersion = "1.0.3"
val farmersDelightCheckout = file("../FarmersDelight")

sourceControl {
    gitRepository(uri("https://github.com/IOVEYOUMC0/Farmersdelight-Plugin.git")) {
        producesModule("com.huidu.farmersdelight:farmersdelight-plugin")
    }
}

if (farmersDelightCheckout.isDirectory) {
    includeBuild(farmersDelightCheckout) {
        name = "farmersdelight-plugin"
    }
}
```

依赖本身随后可以像普通坐标一样写在 `build.gradle.kts` 里。模板改成在 `settings.gradle.kts` 里加，是因为它想让
版本号和通道声明放在同一处：

```kotlin
gradle.beforeProject {
    afterEvaluate {
        if (configurations.findByName("compileOnly") != null) {
            dependencies.add("compileOnly", "com.huidu.farmersdelight:farmersdelight-plugin:$farmersDelightApiVersion")
        }
    }
}
```

`libs/` 下不再放任何 jar，且该目录已被 git 忽略，以免有人再放回一份过期的副本。有测试的附属把同一个坐标
也加到 `testImplementation`。

附属都用 Java 21 和 `options.release.set(21)`，它们的 `compileOnly` CraftEngine 产物照旧从 Maven 取。

## paper-plugin.yml

把 CraftEngine 和 FarmersDelight 声明为必需的服务器依赖，而不是写 `depend` / `softdepend`。你的 `onEnable` 跑起来时，CraftEngine 必须已经定义好物品和方块，FarmersDelight 也必须已经建好 api 背后的各项服务：

```yaml
name: FDAddonTemplate
version: '${version}'
main: com.example.fdaddon.FDAddonTemplate
api-version: '1.21.5'
folia-supported: true
description: Example addon template for the FarmersDelight (CraftEngine) plugin.
authors:
  - YourName

dependencies:
  server:
    CraftEngine:
      load: BEFORE
      required: true
      join-classpath: true
    FarmersDelight:
      load: BEFORE
      required: true
      join-classpath: true
```

`join-classpath: true` 表示你的插件自己解析该依赖的类（直接 import、`Class.forName`，或某个内置桥接代你这么做）。可选联动写在同一段里，用 `required: false`——BrewinAndChewin 就是这样写 `BreweryX` 的。

关于这份文件，几点都来自正在跑的附属：

* `folia-supported: true`：api 的调度封装本身是 Folia 感知的，所以把世界访问都走这些封装的附属，可以名正言顺地声明支持 Folia。
* FDAddonTemplate 和 BrewinAndChewin 都**没有自己的命令**。重载由 FarmersDelight 驱动（`/fd reload all`），配方浏览也由它负责（`/fd recipe book`）。

## onLoad

有两件事必须放在 `onLoad`，赶在 CraftEngine 解析配置之前：

```java
@Override
public void onLoad() {
    // CraftEngine's own api — FarmersDelight does not bridge block-behavior registration.
    if (BuiltInRegistries.BLOCK_BEHAVIOR_TYPE.getValue(Key.of(NS + ":example_block")) == null) {
        BlockBehaviors.register(Key.of(NS + ":example_block"), ExampleBlockBehavior.FACTORY);
    }
    // Copy bundled CraftEngine resources into plugins/CraftEngine/resources/<namespace>/.
    CraftEngineResources.release(this, NS);
}
```

自定义方块行为必须在 `blocks.yml` 被解析之前注册；你内置的 CraftEngine 资源也必须在 CraftEngine 扫描之前 落到磁盘上。方块行为注册走 CraftEngine，资源释放走 FarmersDelight 的公开 API。

## onEnable

下面是 FDAddonTemplate 的 `onEnable`，只保留与生命周期相关的部分。

```java
@Override
public void onEnable() {
    saveDefaultConfig();

    // ALWAYS guard api use with isAvailable(): true only when FarmersDelight is present AND enabled.
    if (!FarmersDelightApi.get().isAvailable()) {
        getLogger().warning("FarmersDelight not available; addon features disabled.");
        return;
    }

    // Count this addon's CraftEngine blocks in FarmersDelight's block-state usage report.
    FarmersDelightApi.get().registerAddonBlockNamespace("fdaddon");

    // Recipe types, buffs, listeners ... (see the other pages). Recipes that must be resolved at runtime
    // are registered from a FarmersDelightWarmupEvent handler, not here.
    FarmersDelightApi.get().registerRecipeType(new ExampleRecipeType());
    getServer().getPluginManager().registerEvents(new ExampleReloadListener(this), this);

    // Folia-safe repeating task; keep the handle so onDisable can cancel it.
    heartbeat = FarmersDelightApi.get().runRepeating(this::onHeartbeat, 20L, 20L * 60L);

    // /fd debugtools integration. Safe even on a non-debug FarmersDelight build.
    DebugToolRegistry.register(new ExampleDebugExtension(this));

    // Refuse PlugMan-style runtime management of this addon.
    getServer().getPluginManager().registerEvents(new PluginManagerGuard(getName()), this);
}
```

### 配方要等 CraftEngine 的物品就绪

CraftEngine 的物品是延迟加载的，在那一轮加载跑完之前，`FarmersDelightItems.create(...)` 会返回 `null`，所以运行时注册的配方不能在 enable 时跑。改由监听 FarmersDelight 的 `FarmersDelightWarmupEvent` 来注册：它在 CraftEngine 建好物品后触发一次，首次启动与每次 `/ce reload` 之后都会再触发。**不要**用 CraftEngine 自己的重载事件——它在 CE 物品建好之前就触发，引用自定义物品的配方会被静默丢弃。同一个监听器也顺手接上 FarmersDelight 的 `FarmersDelightReloadEvent`，这样一条 `/fd reload all` 就能把附属一起同步：

```java
public final class ExampleReloadListener implements Listener {

    private final FDAddonTemplate plugin;

    public ExampleReloadListener(FDAddonTemplate plugin) {
        this.plugin = plugin;
    }

    @EventHandler
    public void onFarmersDelightReload(FarmersDelightReloadEvent event) {
        plugin.reloadAddon(event.getReason());
    }

    @EventHandler
    public void onFarmersDelightWarmup(FarmersDelightWarmupEvent event) {
        // CraftEngine items are built now: register runtime recipes and anything else that resolves them.
    }
}
```

通过 api 注册的配方能扛过 `/fd reload`：FarmersDelight 重载自己的配方文件时会保留外部注册的配方。

## onDisable

把注册过的东西撤掉，api 调用同样用 `isAvailable()` 把门，并取消所有任务句柄：

```java
@Override
public void onDisable() {
    if (heartbeat != null) {
        heartbeat.cancel();
    }
    if (FarmersDelightApi.get().isAvailable()) {
        FarmersDelightApi.get().unregisterRecipeType(NS + ":example");
        FarmersDelightApi.get().unregisterCookingPotRecipe(NS + ":example_stew");
        FarmersDelightApi.get().unregisterCuttingBoardRecipe(NS + ":example_cut");
        CustomBuffRegistry.unregister(exampleBuff);
    }
    DebugToolRegistry.unregister("example_block");
}
```

`DebugToolRegistry` 是静态注册表，活得比你的插件类加载器久，所以反注册不是可选项——留一条陈旧条目，就等于 让一个已经拆掉的 manager 一直可达。

## 重载与热卸载的三道防线

FarmersDelight、CraftEngine 以及建立在它们之上的附属，都会在自己的插件类加载器里留下大量后期绑定的引用： CraftEngine 方块行为、逐区块的方块实体、调度任务、事件 lambda。一旦 `/reload` 或者插件管理器关掉了类加载 器，这些引用就会在无法预测的时间点抛 `NoClassDefFoundError`。这件事没有办法做安全，所以项目的答案是**大声 拒绝**。一共三道机制，附属三道都应该装上。

### 1. JVM 级别的重载守卫

系统属性能挺过插件类加载器的重建，所以同一个 JVM 里的第二次 `onEnable` 是可以被识别并拒绝的。 FarmersDelight 用的是 `farmersdelight.enabled.in.this.jvm`，BrewinAndChewin 照抄了同样的做法：

```java
private static final String RELOAD_GUARD_PROPERTY = "brewinandchewin.enabled.in.this.jvm";

@Override
public void onEnable() {
    if (System.getProperty(RELOAD_GUARD_PROPERTY) != null) {
        getLogger().severe("PLEASE DO NOT /reload OR HOT-DISABLE Brewin' and Chewin'.");
        getLogger().severe("To apply config changes: /stop then start the server again.");
        getServer().getPluginManager().disablePlugin(this);
        return;
    }
    System.setProperty(RELOAD_GUARD_PROPERTY, "1");
    // ... real startup
}
```

属性名自己起。原版 `/reload` 是核心命令，监听器拦不住——这正是除了下面那道守卫之外还需要这一道的原因。

### 2. `PluginManagerGuard`：拒绝 PlugMan 一类的命令

`com.huidu.farmersdelight.api.util.PluginManagerGuard` 是一个开箱即用的 `Listener`，用你自己的插件名 （写在 `paper-plugin.yml` 里的那个）构造。它的公开接口面就这些：

```java
public final class PluginManagerGuard implements Listener {

    public PluginManagerGuard(String pluginName);

    @EventHandler(priority = EventPriority.LOWEST, ignoreCancelled = true)
    public void onPlayerCommand(PlayerCommandPreprocessEvent event);

    @EventHandler(priority = EventPriority.LOWEST, ignoreCancelled = true)
    public void onServerCommand(ServerCommandEvent event);
}
```

你唯一会碰的成员是构造器——那两个 handler 之所以是 public，是 Bukkit 事件系统的要求，不是让你去调的。其余 一切（命令名集合、动作词集合、匹配逻辑、拒绝提示）都是 private。类是 `final`，并且**不带任何** `@ApiStatus` 注解——这点和 `FarmersDelightApi`、`ApiTask`、`DebugToolRegistry` 不同；反正它既没有可继承的东西，也没有可 实现的东西。

`pluginName` 原样存一份用于拒绝提示，另外用 `Locale.ROOT` 转一次小写用于匹配。它没有判空——传 `null` 会在 构造器里抛 `NullPointerException`。老实用 `getName()`。

```java
getServer().getPluginManager().registerEvents(new PluginManagerGuard(getName()), this);
```

FDAddonTemplate 和 BrewinAndChewin 都装了它（ExpandedDelight 没装）。它以 `EventPriority.LOWEST` 且 `ignoreCancelled = true` 监听 `PlayerCommandPreprocessEvent` 和 `ServerCommandEvent`，当下面三条同时成立时 取消命令：

* 命令名（剥掉可能存在的 `namespace:` 前缀之后）属于 `plugman`、`plm`、`pluginmanager`、`plugmanx`、 `plmx`、`pluginsmanager`、`pl`；
* 第一个参数属于 `unload`、`reload`、`disable`、`enable`、`load`、`restart`、`stop`、`start`；
* 后面某个参数与你的插件名大小写不敏感地相等。

`info`、`list` 这类只读动作会被放行。命令发起者会收到一条提示，告诉他改用 `/stop` 重启。原版 `/reload` 不在覆盖范围内——那是上一道守卫的活。

### 3. `RequiredPluginWatchdogListener`：让运行时禁用逐级传导

FarmersDelight 自己装了 `CraftEngineWatchdogListener`：服务器还在跑的时候 CraftEngine 被禁用，它就把自己 也关掉，而不是让每一个绑在 CraftEngine 上的任务狂抛异常。你的附属应该在下一层做同样的事。这是 BrewinAndChewin 的完整实现：

```java
public final class RequiredPluginWatchdogListener implements Listener {

    private static final Set<String> REQUIRED = Set.of("FarmersDelight", "CraftEngine");

    private final BrewinChewinPlugin plugin;

    public RequiredPluginWatchdogListener(BrewinChewinPlugin plugin) {
        this.plugin = plugin;
    }

    @EventHandler(priority = EventPriority.MONITOR)
    public void onPluginDisable(PluginDisableEvent event) {
        if (!REQUIRED.contains(event.getPlugin().getName())) {
            return;
        }
        if (plugin.getServer().isStopping() || !plugin.isEnabled()) {
            return; // normal shutdown, or we are already going down
        }
        plugin.getLogger().severe(event.getPlugin().getName() + " was disabled while the server is running.");
        plugin.getLogger().severe("Restart the server (/stop) to bring both back up.");
        plugin.getServer().getPluginManager().disablePlugin(plugin);
    }
}
```

那两个提前 return 很关键：没有它们，正常关服时这个监听器就会误触发，而那时候所有插件本来就在关。用 `MONITOR` 优先级是因为它只观察和响应，从不修改事件。

### 关于持久化状态的顺序问题

你的附属依赖 FarmersDelight，所以它**先于** FarmersDelight 和 CraftEngine 被禁用。如果你的附属要持久化 方块实体状态，就得在 `onDisable` 的最开头、类加载器还开着的时候把它冲刷或预序列化掉。BrewinAndChewin 的 `onDisable` 第一件事就是调用 `KegChunkListener.passivateLoadedChunks()`，原因正是 CraftEngine 序列化区块 发生在 BAC 已经消失之后，那时再进入酒桶的保存代码会以 "zip file closed" 失败。

## 接下来

* 入口类本身（`apiVersion`、`hasFeature`、`isAvailable`、`isFolia`、`consoleMessage`、`isDebugEnabled`、 `registerAddonBlockNamespace`）见 FarmersDelightApi 一页。
* 调度（`runAtLocation`、`runLaterAtLocation`、`runRepeating`、`ApiTask`）单独成页，包含 Folia 下每个回调 落在哪个线程。
* `/fd debugtools` 接入见调试工具一页。
* 配方、物品、方块、buff、事件、文本、配置辅助各有自己的章节。

## 相关页面

* [FarmersDelightApi 入口](farmersdelight-api.md)
* [事件](events.md)
* [配方包总览](recipes-overview.md)
* [调度](scheduling.md)
