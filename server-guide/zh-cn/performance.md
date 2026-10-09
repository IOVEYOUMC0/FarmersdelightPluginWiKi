---
icon: gauge
---

[English](../en/performance.md)

# 性能排查与调节

先确认成本来自哪个功能，再调整对应参数。`performance.warnings` 只控制告警，提高阈值不会减少计算量；默认预算也不是全服 MSPT 的保证。

## 做一次可比较的采样

1. 记录 Paper/Folia、Minecraft、CE、FD 与附属版本，以及玩家数、已加载工作站数、正在烹饪的数量。Folia 还要记录这些工作站集中在一个区域还是分散在多个区域。
2. 等区块恢复和配方注册完成，确认控制台出现 *Content ready*。分别观察空闲、正常烹饪和密集自动化；测试首次打开配方书时，区分第一次与之后的打开。
3. 玩家持有 `farmersdelight.admin` 权限时执行 `/fd stats profile 600 all`，正常 TPS 下约 30 秒。结束后用 `/fd stats` 查看结果；功能参数和统计上限见[按功能采样性能](troubleshooting.md#按功能采样性能)。
4. 用服务端的 spark 同时观察实际执行线程的调用栈、分配/GC 和 MSPT。Folia 应查看繁忙的区域线程，不能只看全局调度线程。
5. 相同负载重复三次，再只改一个参数，用同样步骤复测。比较每次的最大耗时与 P95，也检查出餐速度、漏斗节奏和显示是否正常。

插件采样的功能行包含下层调用，不能直接相加。它测的是经过时间；网络提交不等于实际发送，各区域计时也不等于全服 MSPT。零调用通常意味着该功能在窗口内没有执行。

## 保存和序列化采样

```text
/fd debugtools profile serialization 600
/fd debugtools stop
/fd stats
```

需要 debug 构建和 `farmersdelight.admin`；普通构建可用 `/fd stats profile 600 serialization`。debug 入口默认 600 tick，允许 20-12000 tick。采样器全服只运行一个会话，记录各区域真实调用，不按发起玩家筛选。

四类结果是实时容器保存、物品 NBT 编码、已接入保存锁的获取耗时和关闭快照的 holder 查询；输出调用数、平均、P95、最大和累计耗时。保存入口覆盖 FD、BAC、CAB、BBQD；停放标签直接写回不进入实时保存计时。物品编码也可能发生在打包等其它已接入路径，锁获取不覆盖外层早已持有的锁。

正常操作并等待自动保存即可；打开 GUI 本身不保证触发持久化。窗口内没有执行的路径显示零调用，需要时延长采样。命令不主动保存、卸载或重载区块，不改库存。停止后仍可用 `/fd stats` 查看结果。各行会嵌套，不能相加为 MSPT，也不包含 CE 整个区块编码、压缩及磁盘写入；配合 spark 定位调用栈和 GC。

## 预算的作用与取舍

以下是新安装的默认值，已有服务器以实际 `plugins/FarmersDelight/config.yml` 为准。修改这些 FD 参数后用 `/fd reload config`；替换插件 JAR 需要完整重启。

| 配置路径 | 默认值 | 调整时的取舍 |
| --- | --- | --- |
| `cooking-pot.tick-budget` | 512 / 每 4 tick | 限制每轮处理的活跃厨锅数；超出时轮转，响应及出餐可能延后 |
| `skillet.tick-budget` | 512 / 每 4 tick | 限制放置煎锅轮询数；超出时可能延缓烹饪 |
| `stove.tick-budget` | 512 / 每 4 tick | 限制炉灶烹饪轮询数；超出时处理和出餐延后 |
| `performance.budgets.reload-recipe-registrations-per-tick` | 32 | 重载完成更慢，每 tick 注册工作更少；0 按 1 处理 |
| `performance.budgets.reload-visual-refreshes-per-tick` | 32 | 重载后工作站外观恢复更慢 |
| `performance.budgets.startup-chunk-loads-per-tick` | 16 | 启动时 FD 已加载区块数据的恢复更慢；不控制所有附属 |
| `performance.budgets.chunk-effect-packet-budget` | 50 | 减少每区块每 tick 的粒子/音效广播；一次广播可发给多人，不能当作总网络包数上限 |
| `performance.budgets.effect-tick-interval-ticks` | 4 | 应调高以减少食物效果更新频率，代价是进度条不够平滑；持续时间仍按间隔扣减 |
| `performance.proxy-display.view-distance` | 64 | 远处工作站显示更早消失 |
| `performance.proxy-display.sync-interval-ticks` | 20 | 应调高以减少可见性检查，代价是接近/离开时显示更新更慢 |
| `performance.proxy-display.sync-batch-size` | 256 | 每轮检查更少的展示物，完整轮转更慢 |
| `skillet.handheld.progress-display.enabled` | true | 关闭后省去周期进度显示刷新，手持烹饪仍可使用 |

厨锅与炉灶按游戏 tick 推进计时，但预算不足仍可能推迟一次完成的处理；煎锅也可能变慢。调低预算后必须同时比较玩法吞吐，不能只看单 tick 耗时下降。

## 按场景排查

| 场景 | 先观察 | 可尝试的调整 |
| --- | --- | --- |
| 密集厨锅、漏斗自动化 | `cooking_pot` 的 P95/最大值；配方匹配和漏斗事件调用栈 | 调整锅预算，降低粒子/音效概率与观察距离；漏斗数量和节奏还受服务端配置及其他插件影响 |
| 大量手持煎锅 | `handheld` 与 `handheld_display`；物品模型和发包分配 | 先关闭周期进度显示再复测；需要停用该玩法时用 `skillet.handheld.enabled` |
| 工作站展示物很多 | spark 中可见性同步的成本与实际网络负载 | 减小显示距离，增加同步间隔，适度减小同步批次 |
| 加入 NBT 精确配方后变慢 | 物品序列化、Base64 与 GC 的调用栈 | 不需要完整物品数据约束时，改用身份/标签食材；需要约束时保留 NBT，避免错误匹配 |
| 启动或重载卡顿 | 配方注册、区块恢复、附属恢复、资源包生成分别耗时 | 调低对应 FD 预算；CE 打包和附属自己的恢复需各自分析 |
| 只在区域集中时变慢 | Folia 上最忙区域的线程 | 分散密集机器并复测；锁与异步任务不能代替 Bukkit 对象的区域归属 |

外部物品应复用 CE 已注册的来源，只配置实际使用的来源，见[经验与外部物品来源](first-config.md)。缺失来源会保留重试以适应 CE 延迟注册；持续的未知来源需要纠正配置。

## 下一步

[迁移与升级 →](migration.md)
