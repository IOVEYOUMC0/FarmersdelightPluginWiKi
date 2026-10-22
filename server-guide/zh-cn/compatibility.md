# 兼容性矩阵

[English](../en/compatibility.md)

本页写明当前构建经过哪些环境验证、与哪些插件集成、以及明确不支持什么。**已验证**表示该路径在记录的轮次里
实际跑过（记录在工作区 `verification-records/`）；**支持**表示代码路径存在、按预期可用，但最近没有重新验证。

## 核心环境

| 项目 | 版本 / 范围 | 状态 |
| --- | --- | --- |
| 服务端 | Paper 或 Folia 1.21.5+ | 支持；Folia 调度按区域归属 |
| Java | 1.21.5–1.21.11 用 21；26.1+ 用 25 | 支持 |
| CraftEngine | 编译目标 26.9.1，声明 26.8.2+ | 支持；符号检查覆盖 26.8.2 / 26.9 / 26.9.1 / 26.9.2 与 26.10 快照 |
| Folia 区域工作（锅 tick、漏斗传输、GUI 写入） | 当前 Folia 构建 | **尚未在真实 Folia 服务端验证**；使用[故障排查](troubleshooting.md#folia-并发压力检查清单)中的手动清单 |
| 资源包 | 必需，由 CraftEngine 生成 | 支持；玩家无需客户端模组 |

## 可选插件集成

| 插件 | 方向 | 提供什么 | 状态 |
| --- | --- | --- | --- |
| PlaceholderAPI | FD → PAPI | `%farmersdelight_buff_*%`、`%farmersdelight_api_version%`、`%farmersdelight_discovery_enabled%`、`%farmersdelight_discovered_*%` | 支持 |
| UltimateAdvancementAPI | FD → 它 | 成就页 UI；无 layout 重载的非补丁版本会禁用 FD 成就页而不是崩溃 | 支持 |
| AuraSkills | FD → 它 | 厨锅技能经验（可配置奖励） | 支持 |
| Leveler | FD → 它 | 经 CE provider 的厨锅经验 | 支持 |
| MMOItems、ItemsAdder、Nexo、Oraxen | FD ← 它们 | 经 CraftEngine `ItemSource` 的外部物品来源；把 provider 名加入 `external-item-sources` 即可识别手持物品 | 支持 |
| WorldGuard | FD → 它 | 总开关 `farmersdelight-use` 与每个工作站一个 flag；附属可注册自己的 flag | 支持 |
| 其他领地/圈地插件（GriefPrevention、Lands、Towny、Residence 等） | FD ← CE | 经 CraftEngine 自带 AntiGriefLib provider 检查，工作站使用与采收尊重领地 | 支持 |
| Kaleidoscope Cookery（森罗物语） | FD ← 它 | 手持记录的菜谱物品右键厨锅自动投料；CE 包声明的 `c:` 成员桥接进 FD 标签注册表 | 支持（仅投料；不向其注册配方或刀具） |
| Brewin' And Chewin'、End's Delight、Crabber's Delight、Barbeque's Delight | FD ← 附属 | 附属内容与工作站通过 FD API 接入 | 支持（各自是独立插件） |
| VillagersDelight | 独立插件 | 农民种植/收割 CE 作物与社交分食 | 支持（非必需） |

## 明确不集成及原因

| 领域 | 现状 |
| --- | --- |
| Bedrock / Geyser 客户端 | 自定义物品、方块、模型只存在于 Java 资源包；基岩玩家看不到。 |
| 客户端模组（JEI/EMI） | 不适用：配方查看器与编辑器是服务端 GUI。 |
| 地图渲染（Dynmap/BlueMap） | 自定义方块是 CraftEngine 状态，不是可渲染的原版方块。 |
| CoreProtect / Prism 日志 | FD 方块变更不经这些插件记录；需要时用其自定义方块支持。 |
| 经济（Vault） | 未实现经济挂点。 |
| 任务插件（BetonQuest/Quests） | 使用 `api.event` 事件；没有预置集成。 |
| MythicMobs / ModelEngine | 模型来自 CraftEngine，没有直接 hook。 |
| 权限插件 | 只使用 `farmersdelight.*` 权限；没有 LuckPerms 专用代码。 |

## 验证策略

* CI 对每个改动执行构建 + 十三项仓库检查（`tools/run_all_checks.py`）。
* 实机轮次记录在仓库外（`verification-records/`），这也是某行从“支持”变为“已验证”的依据。
* Folia 并发清单按设计是手动的；仓库检查无法证明区域归属。
* “支持”行代表按预期可用，但不保证每个版本都重跑。状态列是当前真实状态，不是宣传口径。
