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

`datapacks.tags-enabled` 会把通用物品标签写进主世界的 `datapacks/` 目录，并共享给所有世界；
`enchantments.install-datapack` 安装背刺附魔，`damage-type.install-datapack` 安装炉灶灼烧伤害类型。注册表数据只在
启动时读取一次，所以改动需要重启服务器；`/fd reload enchant` 与 `/fd reload damage` 可重新安装这两个数据包。你
对已安装文件的改动会在插件更新后保留。战利品注入不再是一个开关——它是 FarmersDelight 的 CraftEngine 资源包内容的
一部分，编辑 `vanilla_loots.yml` 后执行 `/ce reload all`。

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

## 下一步

[确认行为已加载 →](verifying.md)
