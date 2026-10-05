
[简体中文](../zh-cn/container-gui.md)
# Container GUI

Package: `com.huidu.farmersdelight.api.gui`

This package is the shared engine behind a container GUI: a character-grid **layout** on the viewer's side and a
flat **store** on the station's side. The recipe book is the same shape — [RecipeBookLayout](recipe-book-layout.md)
extends `GuiLayout` — so a cell means the same thing in both.

Types covered here:

| Type | Kind | `@ApiStatus` |
| --- | --- | --- |
| `GuiLayout` | interface, read-only view | `@OverrideOnly` |
| `GuiLayouts` | static facade, private constructor | — |
| `GuiSlotGroup` | record | — |
| `GuiWriteBack` | static facade, private constructor | — |
| `ContainerGuiKernel` | final, engine | — |
| `ContainerGuiKernel.Controller` | interface the station implements | — |
| `GuiItems` | static facade, decoration builder | — |

## GuiLayout: addressable is not writable

```java
@ApiStatus.OverrideOnly
public interface GuiLayout {
    String BACKGROUND = "background";

    int rows();
    default int size();                                  // rows() * 9
    default boolean contains(int rawSlot);               // 0 <= rawSlot < size()
    boolean isFunctional(int slot);
    default int[] functionalSlots();                     // ascending, fresh array
    int[] slotsOf(String type);                          // ascending, fresh array, empty when unknown/null
    default int firstSlotOf(String type);                // -1 when none
}
```

A cell whose legend type is `background` — or a cell outside the grid — is decoration. Every other cell is
**functional**, meaning it is part of the layout's contents and can be addressed.

**Functional does not mean writable.** A station's display-only cells (a progress bar, the output slot, the meal
preview) are functional and must still never be written back by a GUI. `isFunctional` answers "does this cell
hold contents", not "may the viewer change it"; the write-back decision belongs to `GuiWriteBack` /
`Controller.mayCommit`.

`isFunctional` is the primitive because implementations hold different structures — a station reader answers
from its icon table, the parsed view scans a precomputed cell array. `functionalSlots`, `slotsOf` and
`firstSlotOf` are derived and hand out **fresh arrays**, so a caller may keep, sort or mutate what it gets
without affecting the layout. A hot loop should call `functionalSlots()` once and cache it rather than per tick.
An unknown or `null` type returns an empty array, never `null`.

## GuiLayouts: the generic reader

`GuiLayouts` is the generic half of a `gui.yml` layout reader, with no CraftEngine and no server access: it
reads `rows`, `layout` and `legend`, and its diagnostics are plain data transforms.

```java
public static GuiLayout parse(ConfigurationSection section);       // null section -> null
public static String[] cellTypes(int rows, List<String> layout, Map<Character, String> legend);
public static int warnUnknownCharacters(Logger logger, String configPath, int rows,
                                        List<String> layout, Map<Character, String> legend);
```

`parse` reads rows (default 3, at least 1), `layout` and `legend` (single-character keys), and returns a view
that answers `GuiLayout` **only** — titles, items and station-specific slot roles stay with the caller's own
config class.

`cellTypes` returns the normalised grid: `rows * 9` entries in slot order, never `null`, and a **new array** on
every call. Anything that is not a drawn cell with a usable legend entry reads as `background`:

* a missing row, a row shorter than nine cells, or a row that is `null`;
* a whitespace character;
* a character the legend does not define;
* a legend entry that is `null` or blank.

Rows beyond `rows` and characters beyond column nine are ignored, and a row count below 1 yields an empty
array.

### It does not agree with `GuiConfig.getSlotType` in one place

A station that reads the same grid through its own config class has a second, literal lookup. The two differ in one respect:

* `GuiConfig.getSlotType(slot)` returns `null` for a cell the grid does not draw (a missing row, or a column
  past the end of the row) and returns the legend value **verbatim** — so a legend entry written as a key with
  no value stays `null`;
* `GuiLayouts.cellTypes` / `parse` **normalise**: those same cells and entries become `background`.

They agree on every drawn cell whose legend entry is a non-blank type. They differ exactly where the legend
maps a drawn character to `null` or to a blank string. That is why `RecipeBookLayout.slotType(int)` — the
recipe book's own primitive — normalises the same way `cellTypes` does; all three then agree per drawn cell.

### warnUnknownCharacters

Logs the three grid problems and returns how many it logged:

* the row count disagrees with `rows`;
* a row is not nine characters wide;
* a drawn character the legend does not define.

It is self-contained by design: it takes the logger and a config path instead of reaching for the plugin, so an
addon can report through its own logger with no static lookup and no language key. A `null` logger or layout
logs nothing, and a whitespace character is not a problem. A blank or `null` path is reported as `gui.yml`.

## GuiSlotGroup: one type to one contiguous store range

```java
public record GuiSlotGroup(String type, int storeStart, int count, @Nullable String iconType) {
    public int storeIndex(int offset);      // storeStart + offset
}
```

A container GUI has configured cells on one side and a flat array of stored values on the other; this record is
the join. `type` picks the cells (`layout.slotsOf(type)`), `storeStart` is the store index the first of those
cells maps to, and `count` caps how many of them carry values — a layout that draws more cells of a type than
the store has room for simply leaves the extra cells decorative.

`iconType` names the placeholder painted into a cell whose stored value is empty, or `null` when an empty cell
is just cleared. The item itself comes from the controller (`Controller.icon(iconType)`), because only the
station knows where its placeholder comes from.

## GuiWriteBack: may this snapshot still be committed?

Every container GUI hands each viewer its own snapshot inventory, and vanilla applies a click a tick later, so
a write-back carries a value that may have been painted before the station's ticker ran. Committing it blindly
hands back an item the station already spent, or erases one that arrived meanwhile — **the item exists twice or
vanishes**.

```java
public static boolean mayCommit(@Nullable ItemStack stored, @Nullable ItemStack paintedBaseline);
public static boolean mayCommit(boolean storedEmpty, boolean paintedEmpty, boolean similar,
                                int storedAmount, int paintedAmount);
```

The rule is the only source of write-back policy:

* `null` and air are the same thing (an empty cell);
* **both sides empty** → commit allowed;
* otherwise **the same item in the same amount** (`isSimilar` + equal amounts) → commit allowed;
* everything else → **refused**.

**A refusal is not "keep what the viewer has".** `false` means the caller must repaint that cell from the store;
silently keeping the GUI value is exactly the duplication or loss this guard exists to prevent. The kernel
already does that repaint for you.

The comparison is plain data: it reads no world, clones nothing and runs on any thread. The value-only overload
exists so the rule can be asserted without a running server.

## ContainerGuiKernel

```java
public ContainerGuiKernel(Controller controller, GuiSlotGroup... groups);
public boolean hasLayout();
public boolean fill(Inventory view);
public boolean refreshAll(Inventory view);
public boolean refreshAll(Inventory view, IntPredicate skipPending);
public boolean refreshSlot(Inventory view, int rawSlot);
public boolean syncAll(Inventory view);
public boolean syncSlot(Inventory view, int rawSlot);
public int storeIndex(int rawSlot);                       // -1 when the slot holds no contents
```

The kernel owns the parts every container GUI repeats: mapping configured cells to store indices, remembering
what each cell was painted from, refusing a stale write-back and repainting the refused cell from the store
instead, and the skip rule for edits still in flight.

* `fill` paints the backdrop on every cell the layout does not use, then each group's stored values (a
  placeholder icon where a cell is empty).
* `refreshAll` repaints every configured cell from the store. `refreshAll(view, skipPending)` skips the cells
  `skipPending` accepts — a cell whose edit is still in flight owns the value the player is about to take, so
  repainting it from the store would show the item twice and hand it out again.
* `refreshSlot` repaints one configured cell, restoring the placeholder for an empty one; a cell that is not a
  configured cell is left alone.
* `syncAll` commits every configured cell the guard allows, repaints the cells it refuses, and calls
  `markDirty()` once afterwards.
* `syncSlot` is the single-cell version used by the delayed vanilla click and drag synchronisation. A cell the
  guard refuses is repainted from the store; a slot that is not a configured cell is left alone (and returns
  `true`).
* Every method returns `false` when the station has no layout, so a caller can fall back to a hardcoded grid.
  `storeIndex` then returns `-1`.

### The Controller contract

```java
public interface Controller {
    @Nullable GuiLayout layout();                 // null = every engine method reports "nothing done"
    int slotCount();
    @Nullable ItemStack stored(int storeIndex);   // the authoritative value; must not record anything
    void store(int storeIndex, @Nullable ItemStack item);
    Object storeLock();                           // a real monitor, never null
    void markDirty();

    default @Nullable ItemStack icon(@Nullable String iconType);          // null
    default @Nullable ItemStack backdrop();                               // null
    default boolean isPlaceholder(ItemStack stack, @Nullable String iconType);  // false
    default boolean mayCommit(@Nullable ItemStack stored, @Nullable ItemStack paintedBaseline);
}
```

* `stored` must be a pure read. The kernel calls it for every paint and again inside a commit; recording
  something there (a "last seen" field, a dirty flag) would make painting a store mutation.
* `store` owns cloning and normalisation: it receives whatever the cell held, including a `null` where a
  placeholder icon stood, and must clone what it keeps and store `null` rather than an empty stack. The engine
  never hands the store a live stack.
* `storeLock()` **is really used**: the kernel takes it for every store access and for the per-cell paint
  record. It must be a real object, never `null`, and it is reentrant — the usual caller shape (a click handler
  that wraps its whole read-modify-write in the same lock) is unaffected.
* `markDirty()` is called when a commit changed the store, so the station can mark its block entity dirty. The
  kernel calls it once per `syncAll`, and only on a committed `syncSlot`.
* `mayCommit` defaults to `GuiWriteBack.mayCommit`. Override it only for a station whose commit policy
  genuinely differs (a signed delta, for instance) — **a weaker override is how a container starts duplicating
  items.**

## Threading and the region contract

This is the part that cannot be skipped on Folia.

* `fill`, `refreshAll` and `refreshSlot` only write the **viewer's** inventory, so they must run on the region
  that owns the viewer.
* `syncAll` and `syncSlot` read the store and write it back. They run **under `Controller.storeLock()`**, which
  the kernel takes for every store access and for the paint record — so a caller on the viewer's region cannot
  interleave with a ticker on the store's region.
* **The engine never dispatches.** It never calls a scheduler, never reads a world and never moves work between
  regions. If the viewer and the store are in different regions, arranging that is the caller's job: schedule
  each side with the api's scheduler helpers (`FarmersDelightApi.runAtLocation` / `runLaterAtLocation`, see
  [Scheduling](scheduling.md)). On Folia "viewer region ≠ store region" is the normal case, not an edge case.

A container that ignores this either writes the viewer's inventory from the wrong thread or blocks the store's
region on a cross-region read — which is why the split above is part of the published contract rather than an
implementation detail.

## Shape of a station using it

```java
GuiLayout layout = GuiLayouts.parse(section);            // rows / layout / legend
if (layout != null) {
    ContainerGuiKernel kernel = new ContainerGuiKernel(controller,
            new GuiSlotGroup("bait", 0, 1, "bait_icon"),
            new GuiSlotGroup("catch", 1, 3, null));

    kernel.fill(view);                                   // viewer's region
    // ... later, from the store's region or under storeLock():
    kernel.syncSlot(view, rawSlot);
}
```

`GuiItems.build(section)` / `GuiItems.build(section, placeholders)` build a decoration item from the same
`items:` section shape FarmersDelight's own GUIs use (item id, material, custom model data, item model, name /
lore with MiniMessage and glyph tags, and the translatable `name-key` / `lore-keys` pair), so an addon's
decorations can be localised like the rest of the family's.

## Related pages

* [RecipeBookLayout and opening a book](recipe-book-layout.md)
* [Blocks and stations](blocks-and-stations.md)
* [Scheduling and ApiTask](scheduling.md)
* [Version compatibility helpers](compat-utilities.md)
