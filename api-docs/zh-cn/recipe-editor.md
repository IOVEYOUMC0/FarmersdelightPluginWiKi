---
icon: book-copy
---

[English](../en/recipe-editor.md)

# RecipeEditor、EditableRecipe 与 NumericField

从 `RecipeType.editor()` 返回一个 `RecipeEditor`，你的配方就能通过 FarmersDelight 的通用编辑器 GUI 在游戏内编辑。 GUI 由 FarmersDelight 驱动，**存储归你的附属所有**：编辑器把一个 `EditableRecipe` 草稿交给你去落盘或删除，自己 从不碰你的文件。

内置 `/fd recipe edit pot` 和 `/fd recipe edit board` 会写 FarmersDelight 自己的配方文件，也会回写通过统一 `AddonRecipeFiles` 加载的附属配方文件。数据包提供的配方（`cooking_recipes` / `cutting_recipes` / `special_recipes` 段）没有附属文件可写，编辑器会把修改写进 FD 自己的配方文件，同 id 时 FD 文件优先于数据包。附属若只通过注册 API 提供配方，编辑器会把修改写入 FD 配方文件的 `external-overrides` 覆盖记录；删除覆盖后，附属重新注册的原配方会恢复。这样不猜测附属文件路径，也能让修改在重载和重启后保留。

文件保存到服务器插件数据目录，不会同步修改源码仓库或原 CE 包。FD 厨锅/砧板及 BAC 酒桶使用异步原子文件队列；运行配方刷新独立于玩家是否在线。FD 原生保存反馈等待配方发布，并核对保存后的定义；失败仍保留草稿，文件可能已经写入，应检查日志和实际配方。统一 `AddonRecipeFiles` 与主插件复用高级 Choice/NBT、多工具解析。保存中按钮锁定，旧回调只关闭对应窗口。Esc/取消会保留每位玩家最多八份会话草稿，重新打开同一编辑器继续；退出或重载清除草稿。

## RecipeEditor

```java
import com.huidu.farmersdelight.api.recipe.EditableRecipe;
import com.huidu.farmersdelight.api.recipe.NumericField;
import org.jetbrains.annotations.ApiStatus;
import java.util.List;
import java.util.function.Consumer;

@ApiStatus.OverrideOnly
public interface RecipeEditor {
    List<String> itemSlotLabels();
    List<NumericField> numericFields();
    EditableRecipe load(String id);
    boolean save(EditableRecipe draft);
    boolean delete(String id);
    default boolean saveAsync(EditableRecipe draft, Consumer<Boolean> completion) {
        completion.accept(save(draft));
        return true;
    }
    default boolean deleteAsync(String id, Consumer<Boolean> completion) {
        completion.accept(delete(id));
        return true;
    }
}
```

### itemSlotLabels()

每个可编辑物品槽一个标签，列表长度即槽位数。这些标签会显示在编辑器顶部一个说明物品的 lore 上。

**GUI 最多显示 6 个物品槽**（界面槽位 10–15）。声明超过 6 个标签，多出来的槽既不显示也不会写回：保存时只提交下标 `0 .. min(6, 标签数) - 1`，更高的草稿槽位保持 `load()` 放进去的值。BAC 的酒桶用 5 个：4 个原料加 1 个基础液体容器。 模板用 3 个。

### numericFields()

可编辑的数值按钮，可以为空。它们从槽位 28 开始连续排布，到槽位 48 之前为止，因此最多渲染 20 个；声明更多，尾部就 看不见了。

### load(String id)

构造一份草稿。当管理员是在新建配方时，`id` 为 `null` 或空白 —— 这时返回一份空白草稿。如果你返回 `null`， FarmersDelight 会替你构造 `new EditableRecipe(id, itemSlotLabels().size())`，你的数值默认值就丢了；请返回真正的 草稿。

数值默认值要在这里播种。从未设置过的键在 GUI 里会读成该字段的 `min()`，所以没播种的「烹饪时间」按钮会从最小值起 步，而不是从一个合理的默认值起步。

### save(EditableRecipe draft)

管理员点保存时调用。调用之前，FarmersDelight 已经把 GUI 里的物品槽和结果槽提交进了草稿。你在这里写自己的存储、 重载自己的配方管理器，然后返回 `true`。返回 `false` 表示失败 —— 两种情况 GUI 都会提示玩家，然后关闭界面。

校验也放在这里。模板会拒绝没有 id 或没有结果的草稿：

```java
@Override
public boolean save(EditableRecipe draft) {
    if (draft == null || draft.id() == null || draft.id().isBlank() || draft.result() == null) {
        return false;
    }
    store.put(draft.id(), draft.copy());
    return true;
}
```

`save/delete` 是旧同步入口。默认异步方法仍调用它们，适合内存实现；磁盘存储应覆盖 `saveAsync/deleteAsync`，复用下方文件队列。

### delete(String id)

管理员点删除时调用，传入的是 `draft.id()`，对于一份从未保存过的新配方它可能是 `null` —— 请处理这种情况并返回 `false`。

## EditableRecipe

一份可变草稿。它不是接口，直接构造：

```java
public final class EditableRecipe {
    public EditableRecipe(String id, int itemSlotCount);

    public String id();
    public void setId(String id);
    public int itemSlotCount();
    public ItemStack item(int index);            // 空槽或越界返回 null
    public void setItem(int index, ItemStack item);
    public ItemStack result();
    public void setResult(ItemStack result);
    public int resultCount();
    public void setResultCount(int resultCount); // 钳到 >= 1
    public double number(String key, double fallback);
    public void setNumber(String key, double value);
}
```

构造函数会预先填出 `itemSlotCount` 个空槽。`item`/`setItem` 对下标是安全的：越界读返回 `null`、写则什么都不做， 不抛异常。`setResultCount` 至少钳到 1。

数值是一个 `String` → `double` 的 map，所以要用和 `NumericField` 里一致的键读回来。把键写成常量可以避免经典的 拼写错误：

```java
private static final String COOK_TIME = "cook_time";
private static final String EXPERIENCE = "experience";

@Override
public List<NumericField> numericFields() {
    return List.of(
            new NumericField(COOK_TIME, "Cook time (ticks)", 20, 20, 100000, 0),
            new NumericField(EXPERIENCE, "Experience", 0.5, 0, 100, 1));
}

@Override
public EditableRecipe load(String id) {
    EditableRecipe draft = new EditableRecipe(id, itemSlotLabels().size());
    draft.setNumber(COOK_TIME, 200);
    draft.setNumber(EXPERIENCE, 1.0);
    draft.setResultCount(1);
    return draft;
}
```

关于你在 `save` 里拿到的物品 stack，有一点必须知道：编辑器是**基于副本**工作的。点击玩家背包里的物品，会把一个伪造 的、数量为 1 的克隆放到光标上；把它放进编辑器槽位，存下的就是那个数量为 1 的模板。全程没有真实物品被搬动，因此既不 会刷物品也不会吞物品 —— 但这也意味着 `draft.item(i).getAmount()` 恒为 1、不携带任何信息。如果你的格式里有原料数量， 必须另想办法表达。结果的数量取自 `resultCount()`，而不是 `result().getAmount()`。

### 物品身份与 NBT 的保留

副本虽小，**身份**是完整保留的，而且三种槽的保存策略不同：

* **输入 / 工具槽**保留 CE 或外部来源身份（例如 `itemsadder:myfoods:rice`、`mmoitems:AXE:TEST`），不自动附带外部模板的随机属性。普通未注册物品的额外 NBT 可存为 `{item, nbt}` 并精确匹配；显式配置的快照同样保留。支持标签和或选。
* **结果 / 容器槽**对外部物品保留 `{item, count?, nbt}` 完整快照，包含 MMOItems 数据；加载快照时不重新随机构造。CE 默认物品由定义重建，普通原版物品仅在带额外数据时需要快照。已有只写外部 ID 的配方仍通过 CE 来源重建。

身份优先 CE ID → 旧 `mmoitems:TYPE:ID` → 已绑定 CE 外部来源 → 原版材质。原始外部 ID 保留大小写及冒号。来源配置见[首要配置项](../../server-guide/zh-cn/first-config.md)。

## NumericField

```java
public record NumericField(String key, String label, double step, double min, double max, int decimals) {
}
```

* `key` —— `EditableRecipe` 里的数值键。
* `label` —— 按钮上的显示文字。
* `step` —— 左键加、右键减的步长。
* `min` / `max` —— 每次点击后应用的闭区间钳制。
* `decimals` —— 按钮文字里小数点后的位数；`0`（或更小）按整数渲染。

BAC 酒桶的字段：

```java
@Override
public List<NumericField> numericFields() {
    return List.of(
            new NumericField(FERMENT_TIME, "Ferment time (ticks)", 1200, 1, 100000, 0),
            new NumericField(EXPERIENCE, "Experience", 0.05, 0, 100, 2),
            new NumericField(TEMPERATURE, "Temperature (1 cold..3 normal..5 hot)", 1, 1, 5, 0),
            new NumericField(FLUID_AMOUNT, "Fluid amount (mB)", 250, 1, 100000, 0)
    );
}
```

注意第三个：编辑器没有下拉控件，于是把一个枚举式的取值建模成带钳制的整数字段。这是惯用的绕法。

## 打开编辑器

```java
FarmersDelightApi.get().openRecipeEditor(player, "fdaddon:example", "example_stew");
```

`recipeId` 传 `null` 表示新建配方。当类型 id 未注册、或其 `editor()` 为 `null` 时，这个调用是静默空操作，所以你的 管理命令要靠自己的状态来判断能不能开，而不是靠返回值（它没有返回值）。和所有配方书调用一样，它会打开界面，必须跑在 玩家所在线程上。

权限也要你自己加 —— FarmersDelight 在这里不替你检查。

## 编辑器 GUI 布局（供参考）

编辑器是固定的 54 格窗口，无法重新排版。知道槽位分布有助于你给自己的管理员写文档：

| 槽位      | 内容                            |
| ------- | ----------------------------- |
| 4       | 列出 `itemSlotLabels()` 的说明物品   |
| 10–15   | 可编辑物品槽（按你声明的数量，最多 6 个）        |
| 16      | 结果物品                          |
| 25      | 结果数量按钮（左 +1 / 右 −1）           |
| 28 … 47 | 数值字段按钮，按 `numericFields()` 顺序 |
| 48      | 保存                            |
| 49      | 取消                            |
| 50      | 删除                            |

如果你像模板那样在内存里跨多次打开缓存草稿，就该持有一个稳定的编辑器实例 —— `RecipeType.editor()` 每次打开都会 查询一遍，每次返回新实例会把上一个实例持有的东西全丢掉。

## 异步文件保存契约

`saveAsync/deleteAsync` 返回 false 仅表示未入队；返回 true 后必须调用 completion 一次，参数表示是否成功。完成回调可以在任意线程，FD 的 GUI 会切回玩家调度器。传给实现的是 `EditableRecipe.copy()`，存储层也应复制对象，不要把可变草稿直接存成已保存定义。

`RecipeEditorFiles.saveAsync(file, path, body, preservedFields, reload, completion)` 复用主插件写队列：文件解析、按当前内容合并保留字段、局部修改和原子替换在异步线程；reload 与回调在主插件全局调度器，玩家退出不阻止刷新。body 为 null 时删除条目。无效 YAML 拒绝覆盖；保留字段的读取发生在该文件事务锁内。

```java
import com.huidu.farmersdelight.api.recipe.RecipeEditorFiles;
import java.util.List;
import java.util.Map;
import java.util.Set;

RecipeEditorFiles.saveAsync(file, List.of("keg_recipes", recipeId), body,
        Set.of("result-amount"), recipes::load, completion);
```

reload 必须同步完成自己的配方发布；若只安排未来任务，应自行在实际发布完成后报告成功。主插件不会猜测附属自己的发布生命周期。队列停止或过载时返回 false，GUI 解锁并保留草稿。
