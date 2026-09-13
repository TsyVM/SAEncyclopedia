# Chapter 55 — CJ Character Customisation: Clothing, Hair, Tattoos, and Body Stats

> **Goal of this chapter:** document the appearance system that makes CJ unique among GTA
> protagonists — a five-layer visual assembly (body shape, clothing, hair, tattoos, accessories)
> driven by the `clothes.dat` rule language, the `shopping.dat` economy, and a pair of floating-
> point RPG stats (fat, muscle) that physically deform the character mesh. This is the data-side
> of the system; the runtime assembly and the stat effects on gameplay are the deep-dive pages.

**Subsystem category:** Gameplay — character appearance and RPG stats
**Depends on:** [C26](../C26-Ped-Tables/C26-Ped-Tables.md) (ped data and stat tables),
[C48.2](../C48-Data-Folder-Sweep/02-economy-and-customization.md) (clothes.dat and shopping.dat),
[C30](../C30-Gameplay-Managers/C30-Gameplay-Managers.md) (CShopping runtime),
[C50](../C50-RenderWare-Reference/C50-RenderWare-Reference.md) (RW clump assembly)
**Ties:** [C17](../C17-IFP-Animation/C17-IFP-Animation.md) (animations differ by body shape),
[C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md) (muscle affects speed/strength),
[C45](../C45-Damage/C45-Damage.md) (fat affects fall-damage threshold)
**RE status:** Documented — `clothes.dat` grammar ✅, `shopping.dat` economy ✅,
fat/muscle stat globals 🟡, body-part assembly pipeline 🟡
**Confidence:** ✅ for all `clothes.dat`/`shopping.dat` data (confirmed by `derive_datasweep.py`
8/8) · 🟡 for the runtime assembly internals (from gta-reversed naming, confirmed structurally)
**Data artifact:** [`RE-Data/data/datasweep.json`](../RE-Data/data/datasweep.json)

---

## Deep-dive pages

- [C55.1 — The clothing rule language and assembly](01-clothing-rule-language.md): `clothes.dat`
  in full — the **87 `SETC`** component rules, `HIDE`/`IGNORE`/`EXCLUSIVE`/`CUTS` logic, the
  body-part slot system, how `CClothesBuilder` (🟡) assembles a multi-part DFF from rule outputs,
  and the seven PC clothing shops in `shopping.dat`.
- [C55.2 — Haircuts, tattoos, and accessories](02-hair-tattoos-accessories.md): the three cosmetic
  layers above clothing — hair meshes attached to the head frame, tattoo texture overlays applied to
  the body skin, and the accessory slot (hats, glasses, watches) — each with their `shopping.dat`
  section data and the runtime mechanism.
- [C55.3 — Fat, muscle, and the body-stat RPG system](03-body-stats-fat-muscle.md): the two
  floating-point globals that physically transform CJ's mesh — how fat scales the body and unlocks
  obese animation variants, how muscle scales visible definition and speeds up melee/swimming, how
  both decay over time, and the five gameplay effects (strength, stamina, swimming speed, fall-
  damage threshold, vehicle acceleration bonus) tied to their values.

---

## 55.0 The result first

| Layer | Data source | Key figure | Runtime class |
|---|---|---|---|
| Body shape (fat/muscle) | RPG stat globals | 2 floats: `0.0`–`1.0` | 🟡 `CStats` |
| Clothing models | `clothes.dat` | **87** SETC rules | 🟡 `CClothesBuilder` |
| Hair | `shopping.dat` §Haircuts | **~15** styles | mesh attach to `Head` frame |
| Tattoos | `shopping.dat` §Tattoos | **~50** designs | texture overlay on body skin |
| Accessories | `shopping.dat` §Hats, etc. | several slots | mesh attach to extra frames |

| File | Lines / entries | What it drives |
|---|---|---|
| `clothes.dat` | 87 SETC + 22 HIDE/IGNORE + 20 CUTS + 6 EXCLUSIVE/END | CJ clothing assembly |
| `shopping.dat` | 37 sections, ~560 items | Every purchasable item, all shops |
| Ped body-fat variants | `ped.ifp` animation groups | Walk/run/sprint differ by fat level |

---

## 55.1 Why CJ's appearance is a system, not an asset

Most GTA protagonists are a single fixed model. CJ is assembled at runtime from **independent
parts** combined according to rule-driven data. This has two consequences:

1. **Mix-and-match without clipping** — the `HIDE` and `EXCLUSIVE` verbs in `clothes.dat` ensure
   that when CJ wears a jacket, the torso-shirt underneath is hidden, avoiding z-fighting and
   visible geometry interpenetration. The combinations are governed by data, not pre-baked.

2. **Physical body deformation** — the fat and muscle statistics are not cosmetic flags; they
   affect the underlying mesh scale and the animation group the engine picks. A fat CJ runs with
   a different animation than a lean CJ — the ped animation system
   ([C17](../C17-IFP-Animation/C17-IFP-Animation.md), [C26](../C26-Ped-Tables/C26-Ped-Tables.md))
   selects the group based on the stat value.

The result is a layered pipeline that reads data files at startup, re-evaluates the clothing rules
when the player shops, and produces a single coherent `RpClump` that the renderer draws
([C50](../C50-RenderWare-Reference/C50-RenderWare-Reference.md)).

---

## 55.2 The five-layer assembly (overview)

```
Layer 1: Body shape
    fat/muscle globals   →  mesh scale, body model variant (lean/fat/obese), animation group

Layer 2: Clothing
    clothes.dat SETC     →  per-slot model+texture pair
    HIDE/EXCLUSIVE       →  suppress conflicting layers
    CClothesBuilder (🟡) →  assemble all active SETC slots into one RpClump

Layer 3: Hair
    shopping.dat §Haircuts → selected hair model
    runtime: attach RpAtomic to the 'Head' RwFrame of CJ's clump

Layer 4: Tattoos
    shopping.dat §Tattoos  → selected texture overlay IDs
    runtime: blit tattoo textures onto the body skin RwRaster

Layer 5: Accessories
    shopping.dat §Hats etc. → accessory model
    runtime: attach RpAtomic to the appropriate RwFrame
```

The layers are built on top of one another. Changing clothing triggers a `CClothesBuilder` rebuild
of layers 2–5; changing fat/muscle triggers a body-shape rebuild of layer 1 (and re-evaluates
which animation group the ped uses). Layers 3–5 are lighter: they are mesh/texture attaches rather
than full clump rebuilds.

---

## 55.3 The `clothes.dat` grammar in summary

`clothes.dat` ([C48.2](../C48-Data-Folder-Sweep/02-economy-and-customization.md)) uses five verbs:

| Verb | Count | Role |
|---|---:|---|
| `SETC <bodypart> <slot> <model> <texture>` | **87** | Assign a model+texture to a body slot |
| `HIDE <slot> <triggers-when>` | **12** | Suppress `<slot>` when another slot is active |
| `IGNORE` / `ENDIGNORE` | **11/11** | Suspend rule processing for a block |
| `CUTS <bodypart> <slot> <model> <texture>` | **20** | Cutscene clothing override |
| `EXCLUSIVE` / `ENDEXCLUSIVE` | **3/3** | Mark a group as mutually exclusive |

The body-part slots covered by `SETC` map to the standard `ePedPieceTypes` breakdown
([C45.1](../C45-Damage/01-every-source-is-a-weapon.md)):
- **torso, tshirt, legs, shoes, hair, head, monocle, glasses, hat** (the primary slots)
- Each slot can hold exactly one model+texture pair at a time

A `SETC` line reads: `SETC <bodypart> <slot> <model-name> <texture-name>`. For example:
```
SETC gimpleg  torso  gimptorso  gimp
```
assigns the `gimptorso` model with the `gimp` texture to the torso slot when the "Gimp Suit" outfit
is active. The 87 rules cover all 14 CJ outfit presets and the default-naked base body.

The `HIDE` rules are the overlap suppression: when a jacket occupies the torso slot, the `tshirt`
slot is hidden by a `HIDE tshirt torso` rule, preventing the shirt geometry from protruding through
the jacket. Without these rules, wearing a thick jacket would clip the shirt underneath.

---

## 55.4 The `shopping.dat` economy for CJ appearance

`shopping.dat` ([C48.2](../C48-Data-Folder-Sweep/02-economy-and-customization.md)) holds 37
sections; the CJ appearance sections are:

| Section | Items | Gating |
|---|---|---|
| `Clothes` | All individual clothing items, by shop | respect/sexy required for premium items |
| `Haircuts` | ~15 styles | none (available once the barber is reachable) |
| `Tattoos` | ~50 designs | none |
| `Hats` | hats/glasses/accessories | none |

Each item line is: `<internal-name> <GXT-key> respect <n> sexy <n> <price>`. The `GXT-key` is
the display name ([C19](../C19-GXT-Text/C19-GXT-Text.md)); the `respect`/`sexy` values are the
minimum stat thresholds the player needs to see and buy the item; the `price` is in in-game dollars.

The seven PC clothing shops (BINCO, Victim, eSé, ProLaps, Suburban, ZIP, Ponsonbys) each have their
own sub-section inside `Clothes`, at ascending price points. Ponsonbys is the most expensive and
requires the highest `sexy` stat to access. The sections in `shopping.dat` are what `CShopping`
([C30](../C30-Gameplay-Managers/C30-Gameplay-Managers.md)) loads at startup to populate the shop
inventory.

---

## Key takeaways

- CJ's appearance is a **five-layer runtime assembly**: body shape (fat/muscle) → clothing models
  (87 SETC rules from `clothes.dat`) → hair mesh → tattoo texture overlays → accessories; each layer
  is independently swappable.
- `clothes.dat` is a **rule language**, not a table: the `HIDE`/`EXCLUSIVE` verbs prevent clipping
  between layers so mix-and-match combinations are always coherent.
- `shopping.dat` holds every purchasable item across seven clothing shops, three barbers, and tattoo
  parlours; **respect and sexy stats gate** premium items — the wardrobe is an RPG progression layer.
- **Fat and muscle are floating-point globals** (0.0–1.0 range) that physically scale the body mesh
  and select different animation groups, making body stats a visual and gameplay system, not just
  a number.

**Continue:** [C55.1 — The clothing rule language and assembly →](01-clothing-rule-language.md)

## Engine relationships

- **Data consumed at init:** `clothes.dat`, `shopping.dat` (read by `CGame::Initialise` via
  `CClothesBuilder::Init` and `CShopping::Init` — [C51.1](../C51-Executable-Lifecycle/01-startup-and-init.md))
- **Per-frame:** body stat deformation re-evaluated when stats change (not every frame — only on
  stat delta); clothing rebuilds happen at shop exit.
- **Ties:** [C17](../C17-IFP-Animation/C17-IFP-Animation.md) (animation groups selected by fat level),
  [C26](../C26-Ped-Tables/C26-Ped-Tables.md) (ped stats that include fat/muscle),
  [C30](../C30-Gameplay-Managers/C30-Gameplay-Managers.md) (CShopping runtime for the economy).
