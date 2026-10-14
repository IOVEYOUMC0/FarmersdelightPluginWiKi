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

Expanded Delight（上游作者 `ianm1647`）正在移植，第一阶段作物已完成；食物、工作站和奶酪玩法仍待实现，尚不属于已发布附属。上表列出当前附属，不能把此移植中的工程视为完整可用的 Expanded Delight。

Crabber's Delight 的村民与流浪商人交易单独写在插件目录的 `trades.yml` 中。该附属尚未发布，因此不提供旧版配置迁移或备份。

## 配方与标签

附属的普通原版配方写在各自 CE 资源中；烹饪锅、砧板和仅用于解释机制的卡片由附属在 FarmersDelight API 中注册。
标签解析同时识别原版标签、CE 物品/方块标签和嵌套标签，因此将附属成员加入一个既有 FD 标签后，配方编辑器预览、配方匹配和行为判定会使用相同结果。

## BAC 啤酒与发酵

默认配方按 MIT 正式版 4.5.0 核对。啤酒使用一份小麦、小麦种子、棕色蘑菇和 **1000 mB 水**（四瓶水或一桶水）；保持正常温度，发酵 9600 tick（20 TPS 下约八分钟）。配方书显示液体数量，一瓶水只有 250 mB；原料已填好时也可再次点击补液，原料槽不要放杂物。

可可软糖需 500 mB 奶；淡黄奶酪需 warm、猩红奶酪需 hot。干肉、泡菜和熏鱼无需基础液体。奶酪切出四块，四块重组一轮。没有 Farmer's Respite 默认不提供康普茶发酵配方，服主可添加覆盖。

升级时完整停服更新 JAR，并比较同步现有 `brewinandchewin/configuration/recipes/keg_recipes.yml`、`farmersdelight/cutting_board_recipes.yml` 和 `items.yml`，保留自己的改动。JAR 不会覆盖已有 CE 配置。

## CAB 救生圈

救生圈装备在腿部槽。水中静止时提供有限浮力；横向移动和冲刺游泳时暂停速度修正，停下后恢复漂浮，避免每 tick 的速度包干扰行走。默认潜行也暂停浮力，可主动下潜。`life-preserver` 配置控制开关、强度、上浮限速和潜行行为；飞行、滑翔、载具状态不施加浮力。实际手感及反作弊仍需在服内验收。

## CAB 便签数据

未签署便条使用书与笔载体，保留模型和 16 堆叠；手持在空中右键进入原生编辑，可保存或签署。右键可放置面放置后，未签署和签署便条都只读；要编辑先取回未签署物品。签署便条手持也只读。没有聊天命令或临时换书编辑。

编辑堆叠便条只改一张，其余原物品返还；主/副手都支持。物品已变化或两手定位不明确时拒绝写回，避免覆盖另一张。原生书屏及放置后关闭客户端预测界面的体验仍需真人验收。

正文、标题、作者和签名状态随家具保存。旧签署便条的空保存若遮住源书正文，会恢复正文；明确清空的未签署保存不读回旧文本。未修改便条破坏后保留源元数据，可与相同原物品堆叠。旧纸载体迁移回书与笔，保留正文、模型和数量。

工作台中用一份签名便签加 1～8 格未签名 CAB 便签复制。普通点击取一批，Shift 按空白数量和容量复制；原件保留，复制代数按原版推进。数字键和丢弃不复制，普通纸/书不能代替 CAB 便签。

工作台中用一张**已签署 CAB 便签＋一个原版玻璃瓶**无序合成瓶中信；未签署便签和普通书不适用。可额外放入同一种物品作为礼物，合成时封存并消耗礼物槽的整堆物品，保留其元数据。普通点击取一瓶；带礼物时 Shift 也只封存一瓶，无礼物时按原料和背包容量批量制作。潜行右键开瓶，普通右键投掷；开瓶、落地和入水打捞均保留原便签正文、作者及复制代数。封存的实物始终只回收一次，不受便签池重复分享开关影响。

完整停服更新 JAR，将已部署包的 `crabbersdelight:note.material` 改为 `minecraft:writable_book`，保持其他自定义配置，重建并发送资源包。请验收主/副手保存、放置/破坏堆叠、地上只读正文、签名/复制、区块卸载及重启；服务端数据测试不能代替客户端交互。
