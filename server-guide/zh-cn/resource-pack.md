---
icon: box
---

[English](../en/resource-pack.md)

# 资源包

## 发包的是 CraftEngine，不是 FarmersDelight

FarmersDelight 提供它方块和物品的模型、贴图、语言条目，但它**不**托管也不发送资源包。它把这些文件写进
`plugins/CraftEngine/resources/farmersdelight/`，由 **CraftEngine** 把它们组装成包并发给玩家。

这意味着：

- **生成、托管、发送**包的一切都在 CraftEngine 里配置，文档也在
  **[CraftEngine 自己的文档](https://mo-mi.gitbook.io/xiaomomi-plugins/)** 里。照那边做即可——没有任何
  FarmersDelight 专属项要配。
- 改了 FarmersDelight 的资源文件后，就按 CraftEngine 的方式重新生成包。用 `/ce reload all`（配置**加**包）或
  `/ce reload pack`（只重建包）——单独的 `/ce reload` 只重载配置，**不会**重建包。这和处理任何其它 CraftEngine
  内容的方式完全一样。

## 你真正会碰到的两种故障

### 1. 紫黑色缺失贴图

如果自定义方块和物品显示成经典的紫黑格子，说明**服务端加载没问题，但客户端没用上当前的包**。几乎总是以下之一：

- 你加了或改了资源，但没重新生成包——执行 **`/ce reload all`**（单独的 `/ce reload` 不会重建包）；
- 玩家没接受 / 没下载包，或客户端把「服务器资源包」设成了禁用——检查客户端的服务器设置；
- 卡了一份旧的缓存包——让玩家重进，或清一下他的包缓存。

缺失贴图是**发包**问题。插件本身没事——可以这样验证：`/ce item give <玩家> farmersdelight:cooking_pot` 仍然会给出
一个**名字**正确的物品（名字来自语言数据，贴图来自包）。

### 2. 发包失败 / 玩家从来收不到

如果玩家进服后从没被提示装包，或者下载失败，这是 CraftEngine 的发包配置问题——取决于你怎么设置 CraftEngine：自托管、
外部托管，还是内置托管。这完全是 CraftEngine 的范畴，见 CraftEngine 关于发包的文档。FarmersDelight 没有任何影响发送的
设置。

## 提示框里的物品图标

厨锅的提示框会显示锅里那份菜的图标，酒桶会显示它的饮品的图标。这些图标是各包 `configuration/*.yml` 的 `images:`
里声明的**图片字形**：一个字体字形，贴图就是该物品自己的 16×16 贴图。随包内容用的是：

| 显示位置 | id | 声明在 |
| --- | --- | --- |
| 厨锅的菜 | `<命名空间>:meal_<物品>` | 提供该菜的包 |
| 厨锅的菜本身是原版物品（`beetroot_soup`、`mushroom_stew`、`rabbit_stew`） | `farmersdelight:meal_<材质>` | FarmersDelight 包 |
| 酒桶的饮品（Brewin' And Chewin） | `brewinandchewin:icon_<饮品>` | Brewin' And Chewin 包 |

每条都使用同样的两个度量，它们描述的是 tooltip 行高而不是贴图尺寸：

- `height: 16` 让字形与 16×16 的物品贴图 1:1；
- `ascent: 8` 让它不与上一行重叠；从 9 开始就会盖住上一行；
- tooltip 会在图标行后补一行空行，因为字形的下半部分需要有行可渲染。

条目也可以指向原版贴图而不自带文件，例如 `file: minecraft:item/mushroom_stew.png`。这类条目**必须**写 `height`
（没有 PNG 可供 CraftEngine 测量），而且路径写错是静默的：重载照样成功，客户端只在那一行里显示紫黑缺失贴图。所以
改名或移动食物贴图之后，请进游戏看一眼那个提示框。条目跟着物品 id 走，删掉物品只会留下永远不会被查询的条目。

## 什么时候重跑 `/ce reload all`

只要你动了 `plugins/CraftEngine/resources/farmersdelight/` 下的任何东西——换贴图、调模型、改方块或配方——就跑一次
`/ce reload all`。它会重新读取配置、重建包，让下一次进服（或下一次游戏内刷新包）拿到改动。（单独的 `/ce reload`
只重载配置、跳过重建包，那样贴图或模型的改动就不会送达客户端。）

## 下一步

[首要配置项 →](first-config.md)
