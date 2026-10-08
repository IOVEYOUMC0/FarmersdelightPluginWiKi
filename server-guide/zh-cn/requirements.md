---
icon: list-check
---

[English](../en/requirements.md)

# 环境要求与加载顺序

## 服务端软件

FarmersDelight 支持 **Paper** 和 **Folia**。插件声明了 `folia-supported: true`，使用 Folia 感知的调度封装。大型负载和跨区域交互仍需在你使用的服务端版本上验证。

| 要求 | 取值 |
| --- | --- |
| 服务端 | Paper 或 Folia（或 Paper 分支：Purpur、Pufferfish、Leaf） |
| Minecraft | **1.21.5 或更高**（1.21.x 线） |
| `api-version` | `1.21.5` |
| Java | **21 或更高** —— 插件自身下限；MC 26.x 服务端需要 **Java 25** |

构建使用 Paper **1.21.5** API 编译，插件声明的 `api-version` 是 `1.21.5`，所以 1.21.5 是下限。更新的构建没问题；
低于 1.21.5 不支持。

Java 21 是插件本身的硬要求——jar 编译到 Java 21 字节码级别，旧版 JRE 加载不了。用 CraftEngine 和现代 Paper 本来就
需要的那套 Java 21+ 运行时即可。注意 Paper 要求 MC 26.x 服务端使用 **Java 25**，会直接拒绝 Java 21，所以这类服务端
请用 Java 25 运行。

## CraftEngine 是硬依赖

FarmersDelight 是一个 **CraftEngine 移植**。它提供的每个方块、物品、配方、模型都是以 CraftEngine 内容形式定义的。
`paper-plugin.yml` 的 `dependencies.server` 里列了 CraftEngine（`required: true`、`load: AFTER`、
`join-classpath: true`，即本插件直接解析它的类），这意味着：

- 必须安装 CraftEngine，否则 FarmersDelight 根本不会启用；
- 服务端会**自动**在 FarmersDelight 之前加载 CraftEngine——加载顺序不需要你操心；
- 启动时 FarmersDelight 会等 CraftEngine 把物品和方块解析完，再去注册配方和内容。

选择与你服务端匹配的 CraftEngine 构建。FarmersDelight 默认使用官方 Maven 的 **CraftEngine 26.9.1** API 编译。

- 当前构建引用的 CE 符号已对 **26.8.2、26.9、26.9.1** 核对通过。这说明引用的类与方法存在，不能据此推断所有功能已经过游戏内验证。
- 较早构建有这三个版本的启动核对记录；最新的 CE 经验/物品来源接入和性能改动尚未实机验证。升级正式服前，请检查实际配置的来源、物品数据与跨区域交互。
- CraftEngine 26.9.2 目前用于源码参考，不代表已经实机验证兼容；更旧版本未核对。
- 随包配置已包含 Minecraft 26.3 及以上的世界生成定义，当前构建在真实 26.3 服务端的行为仍待验证。

## 可选集成（软依赖）

以下这些都**不是必须的**。FarmersDelight 在运行时检测它们，只有插件在场时才点亮对应功能。没有它们一切照常。

- **PlaceholderAPI** —— 暴露 `%farmersdelight_buff_...%` 占位符（清单见[自定义 buff](../../api-docs/zh-cn/buffs.md)）。
- **CE 经验 / 物品兼容来源** —— AuraSkills、MMOCore、Jobs 等经验 provider，以及 ItemsAdder、Nexo、MMOItems 等物品来源，复用 CE 已注册的桥。配置与限制见[首要配置项](first-config.md)。AuraSkills raw 奖励仍使用专用接口。
- **UltimateAdvancementAPI** —— 驱动进度页面 UI。
- **领地 / 圈地插件** —— WorldGuard、GriefPrevention、Lands、Towny、Residence 等一大批插件通过随包的保护层被识别，
  于是工作站使用和采收会尊重领地。除了装上圈地插件本身，不需要额外配置。

以上都不改变安装步骤。一个都没有，直接看 [安装](install.md)。

## 不要热重载插件

FarmersDelight、CraftEngine 以及任何附属都持有指向自身类加载器的活引用（方块行为、逐区块状态、调度任务）。
一次 `/reload` 或 PlugMan 式的卸载会在半途拆掉这些引用，进而报错。要应用改动，请**停服再开**（`/stop`），或用下文
[首要配置项](first-config.md) 里说的定向命令 `/fd reload` / `/ce reload`。插件会主动拒绝对自身的热管理，以保护你的
存档数据。

## 下一步

[安装 →](install.md)

**不支持 Spigot / CraftBukkit**，而且从来就不支持：本插件硬依赖的 CraftEngine 本身就是纯 Paper 插件，
FarmersDelight 也大量使用 Paper 独占 API（数据组件 API、Adventure、按实体调度器、Paper 事件）。
从本版本起清单改为 `paper-plugin.yml`，把这个要求在加载时就说清楚，而不是等到运行时报缺类。
