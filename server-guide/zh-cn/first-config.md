---
icon: sliders
---

[English](../en/first-config.md)

# 首要配置项

`plugins/FarmersDelight/config.yml` 的默认值是照着原模组调的，全新安装原样就能玩。本页只列**大多数服主第一天会改**的
那几项，每一项都指向把它讲全的那一页。它不重复整份配置——随包 `config.yml` 里的注释才是参考手册。

改完用 `/fd reload config` 应用（或重启）。**不要**对整个服务器执行 `/reload`。

## 控制台 / 日志语言

```yaml
language: ''
```

留空则跟随 JVM / 系统语言。用 `zh_cn` 或 `en_us` 强制指定。这一项影响的是**服务器日志、控制台文本，以及没有玩家上下文
的消息**——各个玩家的界面和聊天仍按各自的客户端语言显示。控制台语言的解析规则见 [故障排查](troubleshooting.md)。

## Buff 显示（boss 血条）

```yaml
buff:
  enabled: true
  display:
    enabled: true
    channels:
      - bossbar
    layout-mode: stacked
    styles:
      nourishment: {color: GREEN, overlay: PROGRESS}
      comfort:     {color: BLUE,  overlay: PROGRESS}
```

控制饱足 / 舒适 buff（以及任何附属 buff）怎么显示。如果 boss 血条区域已经被别的插件占用，把 `channels` 换成
`actionbar` 或 `tab_footer`；或者设 `display.enabled: false`，保留效果但隐藏血条。详见[自定义 buff](../../api-docs/zh-cn/buffs.md)。

## 效果：饱足 / 舒适（Nourishment / Comfort）

```yaml
events:
  - on: consume
    functions:
      - type: farmersdelight:nourishment
        duration: 180
```

食物与 Buff 的关联现在写在对应物品的 CraftEngine 配置中。使用 `farmersdelight:nourishment` 或
`farmersdelight:comfort` 作为 `on: consume` 函数，`duration` 单位为秒。内置关联位于
`plugins/CraftEngine/resources/farmersdelight/configuration/items.yml`；全局 Buff 显示、持久化和舒适回血参数仍在
`plugins/FarmersDelight/config.yml`。

## 漏斗交互

```yaml
hopper-interactions:
  enabled: true
```

漏斗 ↔ 工作站桥接的总开关（烹饪锅、砧板、煎锅）。如果漏斗和你别的插件打架，**先把这个总开关关掉**隔离问题，再重新开启
并调各工作站自己的 `allow-hopper: true/false`（在 `config.yml` 里）。

CrabbersDelight 的蟹笼和蚯蚓箱使用原生漏斗容器，不受上面的 FD 工作站开关控制。上方朝下的漏斗输入鱼饵/堆肥，下方漏斗抽取产物；数量与冷却遵循服务端配置（默认每次 1 个、8 tick），红石锁定和库存移动事件生效。钓具箱家具不支持漏斗。

## 烹饪经验与 CE 技能来源

```yaml
experience-reward:
  mode: levelers
  levelers:
    - plugin: mmocore
      target: cooking
      multiplier: 1.0
```

`plugin` 使用 CE 注册的名称，`target` 按目标插件的实际技能、职业或工作 ID 填写，保留大小写和分隔符。示例中的 `cooking` 需要在 MMOCore 中存在，它是 profession/collection skill，不是角色主等级。支持哪些插件以 CE 当前注册结果为准；参考 CE 26.9.2 包括 `auraskills`、`aureliumskills`、`ecojobs`、`ecoskills`、`jobs`、`mmocore`、`mcmmo` 和 `minecraft`。

| mode | 奖励 |
| --- | --- |
| `vanilla`（默认） | 原版经验球 |
| `auraskills` | 旧 `auraskills` 配置中的奖励 |
| `levelers` | `levelers` 列表中的 CE 奖励 |
| `both` | 经验球 + 技能奖励；列表有 `plugin: auraskills` 时，它替代旧 AuraSkills 配置 |
| `none` | 关闭奖励，仍发出只读结算事件 |

每项可设 `amount` 固定量（优先于倍率）、`multiplier`（配方基数 × 倍率）和 `chance`（0–1，或大于 1 的百分数）。基数须为有限正数，数量不能为负或无穷。缺 provider 或目标不可用时不替换成其他奖励。

厨锅在玩家从**成品输出槽成功取出后**结清累计配方批次经验；不是按本次取出的数量分摊。手持容器直取待盛装食物、装碗搬运和漏斗不领取经验。台账会保存，结算后清空并标脏；破坏/移除时先释放未领原版经验，再打包物品，不发技能奖励。缺少台账的旧存档不能从剩余食物份数还原批次经验。固定量和概率按一次结算计算。酒桶等使用 FD 公共经验 API 的附属共用该配置，砧板没有配方经验基数。普通 AuraSkills 奖励经 CE；旧 `raw: true` 保留直接添加 raw XP 的语义。CE 的 `minecraft` provider 直接给玩家整数经验，与 `vanilla` 的小数概率舍入经验球不同；`both` 中再配它会同时获得两个通道。

经验球在工作站区域生成，技能奖励和厨锅/API 结算事件在玩家实体调度器执行。CE 接口复用不保证第三方插件本身兼容 Folia。

## 外部物品来源

CE 已注册的 `ItemSource` 可直接用于 FD 配方输入、工具、容器和结果：例如 `itemsadder:myfoods:rice`、`nexo:rice_knife`、`mmoitems:AXE:TEST`。第一个冒号前是来源名，余下是原始物品 ID；不要把冒号改为下划线，或把原始 ID 全转小写。

配方引用的来源会自动绑定。要在尚无相关配方时识别编辑器中的手持物品，可在 FD `config.yml` 添加：

```yaml
external-item-sources: [itemsadder, nexo, oraxen]
```

MMOItems 始终保留旧 `mmoitems:TYPE:ID` 识别。身份优先 CE ID、旧 MMOItems 数据，然后已绑定外部来源（按来源名排序），最后原版材质。外部物品不会匹配其载体的原版 ID 或原版标签。标签须由 CE 内容声明或 FD 标签 API 注册；CE 的 `recipe.ingredient-sources` 只管理 CE 自己的配方。

来源或模板缺失时构造返回 `null`，不会替换为同材质原版物品。编辑器输入/工具以真实身份匹配，结果及需要保留数据的容器保存完整快照；已有仅 ID 的 MMOItems 配方继续由 CE 构造。当前支持加载时生成的固定模板，不在每次制作时构造玩家专属或随机属性结果。修改来源配置后用 `/fd reload all`；CE 重载也会清除旧绑定和图标缓存。

## 配方发现（锁定配方书）

```yaml
recipes:
  discovery:
    enabled: false
```

默认关闭——每个配方在书里立刻可见。开启后，配方在 FarmersDelight 自己的烹饪锅 / 砧板查看器里以**锁定**状态起步，随玩家
解锁逐人揭示。锁定只影响书的显示，从不阻止在工作站实际制作。`locked-display`、`unlock-on-obtain`、`notify` 见管理员
Wiki（以及开发者文档的 *配方发现* 页）。

> 配方的解锁进度以**配方 id** 为键。重命名配方 id 会重置它的发现进度。见 [迁移与升级](migration.md)。

## 随包安装的数据包

```yaml
datapacks:
  tags-enabled: true
enchantments:
  install-datapack: true
damage-type:
  install-datapack: true
```

Paper 启动时发现并加载 `plugins/FarmersDelight/datapacks/` 内的受管数据包；这三个开关保持原义。标签来源是 `common-tags.yml`，背刺设置仍在 `config.yml`，炉灶伤害与伤害标签在 `damage-types.yml`。BAC 的心脏骤停、CAB 的落椰伤害分别由所属插件自己的 `damage-types.yml` 管理。

旧世界目录中的受管包在迁移这次启动仍正常加载，随后迁入插件受管目录，复制成功后清理旧目录，不再自动生成备份。没有伤害 YAML 时会导入旧 JSON；已有 YAML 时以 YAML 为准。下次完整启动切换到插件目录。注册 ID 保持不变，不要改名已有世界使用的附魔或伤害类型。

`datapacks/` 内的 JSON 是生成产物，修改配置后需完整重启。`/fd reload enchant`、`/fd reload damage` 可重新生成文件，但不会使新注册内容在本次运行生效。FD 与 VD 不再主动调用服务器数据重载；新增附属原版标签成员仍需下一次启动。此前已有的备份目录不会自动清理。

战利品注入仍属于 CE 资源包，编辑 `vanilla_loots.yml` 后执行 `/ce reload all`。

## 进度

```yaml
advancements:
  enabled: true
```

设 `enabled: false` 关掉整棵进度树。保持开启时，`auto-disable-missing: true` 会隐藏那些你删掉了 CraftEngine 内容的
进度，让进度树仍可完成。背景见[迁移与升级](migration.md)与[进度](../../api-docs/zh-cn/advancements.md)。

## 更深的开关在哪

性能预算（`performance.warnings` / `performance.budgets` / `performance.proxy-display`）、粒子 / 音效、显示偏移、热源和
自定义物品的 `container-returns` 位于 `config.yml`。砧板上每个物品 / 标签的显示覆盖表位于
`plugins/FarmersDelight/display-overrides.yml`（`items` / `tags`）。稻草掉落规则的启用白名单位于
`plugins/FarmersDelight/drops.yml`，而**生物的小刀额外掉落已经是包内数据**：写在 CraftEngine 资源包的
`vanilla_loots.yml` 里（条目名形如 `farmersdelight:ham_from_pig`），和包内其它掉落一起调整。
村民 / 流浪商人交易位于 `plugins/FarmersDelight/world-data.yml`，删掉其中一个条目即可禁用该交易。堆肥、熔炉燃料、宠物食物和食物 Buff
关联位于 CraftEngine 物品配置中。CE 配置有意保持无注释，字段说明见[方块行为配置](block-behaviors.md)。第一天基本用不到。

性能预算的默认值、调节步骤和取舍见[性能排查与调节](performance.md)。

## 下一步

[确认行为已加载 →](verifying.md)
