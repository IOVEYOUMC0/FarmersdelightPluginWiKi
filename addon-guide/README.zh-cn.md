---
icon: puzzle-piece
---

# 附属指南

[English](README.md)

FarmersDelight 的附属是独立加载的插件。除 Villagers' Delight 外，它们都依赖 FarmersDelight 与 CraftEngine，并通过
CraftEngine 资源提供物品、方块、配方和标签，通过 FarmersDelight API 注册工作站配方、进度及需要保存状态的玩法。

各附属的署名与许可按下表逐一列出（以上游项目自己的声明为准）：

| 附属 | 上游模组 | 作者 | 上游许可 |
| --- | --- | --- | --- |
| FarmersDelight（本核心插件） | Farmer's Delight | vectorwing | MIT |
| Brewin' And Chewin'（BAC） | Brewin' And Chewin' | ProbablyEyes（所有者）、Umpaz、MerchantCalico、RaymondBlaze、Farcr | MIT，`Copyright (c) 2022 Umpaz` |
| Barbeque's Delight（BBQD） | Barbeque's Delight | MaoMao、lcy0x1 | MIT（`LICENSE`，`Copyright (c) 2024 MaoMao`）；上游 `mods.toml` 另写 LGPL-2.1，见下注 |
| Crabber's Delight（CD） | Crabber's Delight | AlabasterLeking | MIT，仅由上游构建元数据声明 |
| End's Delight（ED） | End's Delight | FoggyHillside | MIT，`Copyright (c) 2022 FoggyHillside` |
| Villagers' Delight（VD） | ——（本项目原创） | 本项目 | 不适用（没有上游模组） |

其中三条需要说明：

* **BAC** —— 本移植的 CraftEngine 内容所依据的那一版 `LICENSE` 写的是「MIT License，Copyright (c) 2022 Umpaz」，该许可随本移植保留
  （随扩展 jar 的 `NOTICE.txt` 一同分发）。上游已于 **2026-08-24（提交 `ed58394`）删除其 `LICENSE` 文件**，之后的版本不再附带它；
  本移植仍按其构建时所依据的 MIT 版本分发。上表的作者串是两个来源的并集，并采用规范拼写：`MerchantCalico` 与 GitHub 账号
  `MerchantPug` 是同一人；`Probleyes`（jar 内 `NOTICE.txt` 的写法）是 `ProbablyEyes` 的旧拼写。
* **BBQD** —— 上游仓库 `LICENSE` 为 MIT（`Copyright (c) 2024 MaoMao`），Modrinth 也把该模组标为 MIT，而其
  `META-INF/mods.toml` 声明的是 `license="LGPL-2.1"`。其移植按仓库 `LICENSE`（MIT）分发。两处上游声明互相矛盾，属上游自身
  的不一致，不是这里的选择。
* **CD** —— 上游**没有 `LICENSE` 文件、也没有版权行**；MIT 仅由其构建元数据声明（`gradle.properties: mod_license=MIT License`、
  `mod_authors=AlabasterLeking`，以及 `neoforge.mods.toml` 中对应的占位引用），因此应视为元数据声明，而非签署过的许可文本。

## 共同加载规则

1. 先安装 CraftEngine 与 FarmersDelight，再安装附属。
2. 修改附属 CE 资源后执行 `/ce reload all`；修改附属 `config.yml` 后执行 `/fd reload all` 或使用该附属自己的重载命令。
3. CE 配置是无注释的纯数据。方块列表、CE 标签和行为字段的写法见[方块行为配置](../server-guide/zh-cn/block-behaviors.md)。
4. 附属物品可加入 FarmersDelight 的标签来复用通用规则，例如 `farmersdelight:milk`、`farmersdelight:compost_accelerant`。

## 内容概览

| 附属 | 主题与核心内容 | 主要配置 |
| --- | --- | --- |
| Brewin' And Chewin' | 酒桶发酵、奶酪熟成、饮酒效果、冰箱 | `keg`、`booze-effects`、`temperature` |
| End's Delight | 末地食物、末地炉灶、龙腿、末地小刀 | `mob-drops`、`dragon-tooth-knife` |
| Crabber's Delight | 捕蟹笼、蚯蚓箱、钓具、椰子、瓶装便条 | `config.yml` 中的 `crab-trap`、`notes`、`tackle-box`；交易见 `trades.yml` |
| Barbeque's Delight | 烤架、食材盆、烤串调味、托盘 | `grill`、`ingredients-basin`、`seasonings` |
| Villagers' Delight | 让农民村民识别和种植 CE 作物 | `crops`、`extra-soils`、`harvest-drops` |

Expanded Delight（上游作者 `ianm1647`）：**还没有移植仓**（`Reference/` 下只有脚手架），因此**本指南不覆盖它**。
本指南不为它作任何许可主张；上表与本节都不含它的许可条目。

Crabber's Delight 的村民与流浪商人交易单独写在插件目录的 `trades.yml` 中。该附属尚未发布，因此不提供旧版配置迁移或备份。

## 配方与标签

附属的普通原版配方写在各自 CE 资源中；烹饪锅、砧板和仅用于解释机制的卡片由附属在 FarmersDelight API 中注册。
标签解析同时识别原版标签、CE 物品/方块标签和嵌套标签，因此将附属成员加入一个既有 FD 标签后，配方编辑器预览、配方匹配和行为判定会使用相同结果。
