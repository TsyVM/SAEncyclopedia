# C48.2 — Economy and customization

Four files hold the game's *content* rather than its engine config: how CJ dresses, what everything costs,
how interiors are furnished, and the licence-plate palettes. Two of them (`clothes.dat`, `shopping.dat`) are
substantial and worth reading closely.

## `clothes.dat` — a clothing rule language

`clothes.dat` is not a table; it is a small **rule language** for CJ's appearance. The verbs (counted by
`derive_datasweep.py`) are:

| Verb | Count | Role |
|---|---:|---|
| `SETC` | **87** | set a clothing component: a body part gets a model + texture |
| `HIDE` | 12 | hide a component when another is worn |
| `IGNORE` / `ENDIGNORE` | 11 / 11 | begin/end an ignore block |
| `CUTS` | 20 | cutscene-specific clothing override |
| `EXCLUSIVE` / `ENDEXCLUSIVE` | 3 / 3 | mutually-exclusive clothing group |

A `SETC` line reads `SETC <bodypart> <slot> <model> <texture>` — e.g. `SETC gimpleg torso gimptorso gimp`
dresses the torso for the "gimp" outfit. The `HIDE`/`IGNORE`/`EXCLUSIVE` verbs are the *logic*: some garments
hide others (a jacket hides the torso beneath), some combinations are mutually exclusive, and cutscenes force
specific outfits (`CUTS`). So CJ's wardrobe is a rule system — 87 component rules plus the constraints that
keep the combinations coherent — which is why the game can mix-and-match tops, bottoms, shoes and accessories
without clipping: the `HIDE`/`EXCLUSIVE` rules enforce it. This is the data behind the clothing-shop
customization, and it is genuinely a *language*, the same shape as [C24](../C24-Surfaces/C24-Surfaces.md)'s
enumerated grammars.

## `shopping.dat` — the whole economy

`shopping.dat` is the single largest gap closed here: **1,359 lines** holding every price in the game, in a
nested `section` tree (**37** sections). The top level is `section prices`, and under it every purchasable
category:

```
section prices
  section CarMods       ; vehicle mod prices (name, GXT key, respect/sexy req, price)
  section Clothes
  section Haircuts
  section Tattoos
  section Food
  section Weapons
  section Furniture
  section Gifts
  section Property
  ... (BINCO, PROLAPSE, ZIP, ammun1..4, Lowriders, Racers, ...)
```

Each item line is `<internal-name> <GXT-key> respect <n> sexy <n> <price>` — so an item carries its display
name (a [C19](../C19-GXT-Text/C19-GXT-Text.md) GXT key), the **respect** and **sexy** stat requirements to
unlock it, and its **cost**. This is precisely the data that fills
[C30](../C30-Gameplay-Managers/C30-Gameplay-Managers.md)'s `CShopping` runtime — that chapter sized the
560-item ledger and the section record; `shopping.dat` is where those items and prices come from. The
`respect`/`sexy` gating in the file is the economic expression of the RPG stats: some clothes and cars are
locked until CJ is respected or attractive enough. So the whole in-game economy — mod garages, clothing
shops, barbers, tattoo parlours, restaurants, Ammu-Nation, real estate — is this one file, section by
section.

## `furnitur.dat` — furnishing interiors

`furnitur.dat` procedurally furnishes interiors. It is a `GROUP` / `SUBGROUP` / `ITEM` hierarchy — **6**
top-level groups (`IT_SHOP`, …):

```
GROUP: IT_SHOP
  SUBGROUP: SHOP_UNIT1_L  1 1 1 1 0 0 0
    ITEM: shlf4_cablft  1  0  100  0    ; model, slot, chance/params
    ITEM: CJ_SS_1_L     2  0  100  0
```

A group is a *kind* of interior (a shop), a subgroup a furnishing unit within it (a shelf run), and the items
the props placed there. This is how the game dresses generic interior shells with appropriate furniture
without hand-placing every object — the interior-generation counterpart to the procedural-object system.

## `numplate.dat` — plate palettes

The smallest of the four: `numplate.dat` holds alternative colour **palettes** for the car licence-plate
character sets, in PaintShop Pro's JASC `.PAL` text format (a comment header, then RGB triples). It tints the
plate glyphs ([C23](../C23-Fonts-HUD/C23-Fonts-HUD.md)-adjacent — a small glyph atlas), giving regional plate
colour variety. Documented for completeness; a palette file, not a system.

## Key takeaways

- `clothes.dat` is a **rule language** (87 `SETC` component rules + `HIDE`/`IGNORE`/`EXCLUSIVE`/`CUTS` logic),
  not a table — the data behind CJ's mix-and-match wardrobe.
- `shopping.dat` is the **entire economy** — a 37-section price tree (cars, clothes, haircuts, tattoos, food,
  weapons, property…) with per-item GXT name, respect/sexy gating and cost — and it fills
  [C30](../C30-Gameplay-Managers/C30-Gameplay-Managers.md)'s `CShopping` ledger.
- `furnitur.dat` (6 `GROUP`s) furnishes interiors procedurally; `numplate.dat` is plate-glyph colour
  palettes.

**Continue:** [C48.3 — Extensions and the binary grid →](03-extensions-and-binary-grid.md)
