
[简体中文](../zh-cn/cooking-pot.md)
# The Cooking Pot

The **Cooking Pot** is the heart of Farmer's Delight. It simmers ingredients over a
heat source and turns them into hearty meals, stews and soups that restore far more
hunger than raw food — and many of them grant a helpful effect while you digest.

## Crafting a Cooking Pot

Shaped recipe (crafting table):

```
[ brick ] [ wooden shovel ] [ brick ]
[ iron  ] [ water bucket  ] [ iron  ]
[ iron  ] [ iron ingot    ] [ iron  ]
```

- `B` = `minecraft:brick` (the item, not the block)
- `S` = `minecraft:wooden_shovel`
- `W` = `minecraft:water_bucket`
- `I` = `minecraft:iron_ingot`

Result: `farmersdelight:cooking_pot` ×1. You get your **empty bucket back** — the
water bucket leaves a `minecraft:bucket` in the crafting grid when the pot is made.

## Placing and heating it

1. Place the pot on top of a **heat source**. The pot cooks only while a heat source
   sits directly beneath it.
2. Right-click the pot to open its cooking interface.

**What counts as a heat source** (the block directly below the pot):

- A lit **Stove** (`farmersdelight:stove` with its fire lit)
- A lit **Campfire** or **Soul Campfire** (must be lit)
- `minecraft:magma_block`
- `minecraft:lava` or a **Lava Cauldron**
- `minecraft:fire` / `minecraft:soul_fire`
- Any block a server admin has added to the `farmersdelight:heat_sources` tag or the
  heat-source list in `config.yml`

A **hopper** counts as a "conductor": if a hopper sits directly under the pot, the
heat source one block further down (below the hopper) still heats the pot. This lets
you extract finished meals below while heating from further down. Feed ingredients from above and containers such as bowls from the sides.

## The cooking interface

When you open the pot you get a grid of slots:

- **Ingredient slots (6)** — drop the items a recipe calls for here.
- **Meal slot** — the finished meal appears here once cooking completes.
- **Container slot** — place a stack of the container a recipe needs (usually a
  `minecraft:bowl`) here.
- **Output slot** — when a meal needs a container, the served meal is pushed here so
  you can pull it out with its bowl.

### How cooking works

1. Put a valid combination of ingredients into the ingredient slots.
2. Keep a heat source lit below the pot.
3. After the recipe's cook time the ingredients are consumed and the meal appears in
   the meal slot.

Some meals are served **in a container**. For example, most stews and soups are
served in a bowl. If the meal needs a bowl:

- Put bowls in the container slot, and the finished meal will be handed out with a
  bowl into the output slot, **or**
- Simply right-click the pot while **holding a bowl** — one bowl is consumed and the
  meal drops straight into your inventory.

Meals that don't need a container (for example a block of cooked food) can be taken
directly out of the meal slot.

Default cook time is 200 ticks (10 seconds); individual recipes set their own time.

## The recipe viewer and the "craftable only" filter

You don't have to memorise recipes. Open the recipe viewer with:

```
/fd recipe
```

This lists every cooking pot recipe (and cutting board recipe) the server has. Click
a recipe to see its ingredients, its container and its result.

There is a **filter toggle** in the viewer:

- **Showing all recipes** — every recipe is listed.
- **Showing craftable recipes** — only recipes you can actually make right now are
  shown, based on the ingredients in your inventory (plus whatever is already inside
  the pot you opened the viewer from).

> **Note:** the "craftable only" filter shows exactly the recipes you currently have
> the ingredients for.

When you open the viewer from a cooking pot, selecting a recipe can also pull the
needed ingredients out of your inventory and into the pot for you.

## Getting the meal out

- **Container meals (stews/soups):** right-click with a bowl in hand, or keep bowls in
  the container slot and collect from the output slot.
- **Other meals:** take them straight from the meal slot.

Leftover containers from some recipes (like a bowl or a bottle) are returned to you
rather than destroyed.

A successful player take from the **output slot** settles all accumulated recipe batches, rather than prorating XP by the servings taken. Later takes cannot claim those batches again. Hand serving from the pending meal, moving meals into bowls, and hopper extraction do not claim XP. The ledger is saved with the pot. Breaking the pot releases unclaimed orbs when the server's vanilla reward channel is enabled; it does not grant skill XP. The server owner chooses the [reward channels](../../server-guide/en/first-config.md).

## Comparator output

A cooking pot emits a **redstone comparator signal**. The strength scales with how
full the pot is across all its slots — an empty pot reads 0, and any content lifts the
signal, rising as the pot fills. This lets you automate a pot with hoppers and
comparators.

## Tips

- Keep the heat source lit — the pot pauses if the fire goes out.
- Hoppers above feed ingredients, hoppers at the sides feed containers, and a hopper below pulls finished output for a simple auto-kitchen.
- Use `/fd recipe` and the craftable filter to plan meals from what you already have.
- Meals restore much more hunger and saturation than raw ingredients, and many give a
  short buff — cooking is almost always worth the effort over eating raw.
