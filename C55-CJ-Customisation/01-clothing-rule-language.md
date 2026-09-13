# C55.1 — The Clothing Rule Language and Assembly

> **The one-sentence version:** `clothes.dat` is not a lookup table but a declarative rule language
> — 87 `SETC` slot assignments plus `HIDE`/`IGNORE`/`EXCLUSIVE`/`CUTS` logic that enforces
> layer coherence — and at runtime `CClothesBuilder` (🟡) evaluates these rules to assemble the
> active set of body-part models into a single coherent `RpClump` that the renderer draws.

**Subsystem category:** Gameplay — CJ clothing assembly
**Depends on:** [C48.2](../C48-Data-Folder-Sweep/02-economy-and-customization.md) (clothes.dat
grammar confirmed), [C30](../C30-Gameplay-Managers/C30-Gameplay-Managers.md) (CShopping for the
shop front-end), [C50](../C50-RenderWare-Reference/C50-RenderWare-Reference.md) (RW clump assembly)
**RE status:** Documented — `clothes.dat` grammar ✅ (`derive_datasweep.py` 8/8); assembly
runtime internals 🟡 (gta-reversed class naming, structure inferred)
**Confidence:** ✅ for all `clothes.dat` counts and grammar (87 SETC, 12 HIDE, etc.) ·
🟡 for `CClothesBuilder` class internals and RW attach mechanism

---

## 1. The `clothes.dat` grammar in full

`clothes.dat` is read once at `CGame::Initialise`
([C51.1](../C51-Executable-Lifecycle/01-startup-and-init.md)) by the clothing subsystem. Its
grammar uses five verbs:

### 1.1 `SETC` — the slot assignment verb (87 instances)

```
SETC  <bodypart>  <slot>  <model-name>  <texture-name>
```

`SETC` is the primary definition statement: it assigns a clothing component to a body-part slot.
A body-part slot is a named attachment point on CJ's body (`torso`, `tshirt`, `legs`, `shoes`,
`hair`, `head`, `hat`, `glasses`, `watch`, `extra1`, `extra2`). The `model-name` is the DFF
basename; the `texture-name` is the TXD entry that textures it.

**Example from the file:**
```
SETC gimpleg     torso    gimptorso      gimp
SETC suit1       torso    suit1torso     suit1
SETC suit1       legs     suit1legs      suit1
SETC tracktop1   torso    tracktop1      tracktop1
```

The 87 SETC rules cover:
- **14 complete outfit presets** (the Gimp Suit, Street Outfit, Cowboy Outfit, etc. — each
  outfit is a set of SETC rules, one per occupied slot)
- **Individual items** (individual shirts, trousers, shoes, hats — the mix-and-match wardrobe)
- **The default "naked" base** (the bare-skin model that shows when no clothing slot is filled)

A slot with no active `SETC` rule shows its default model (the skin). A slot with an active rule
shows the assigned DFF, textured with the assigned TXD.

### 1.2 `HIDE` — the overlap suppression verb (12 instances)

```
HIDE  <slot-to-hide>  <when-slot-is-active>
```

`HIDE` prevents two overlapping geometry layers from z-fighting. When the `<when-slot-is-active>`
slot has an active `SETC` assignment, the `<slot-to-hide>` slot is suppressed entirely — its model
is not attached to the clump.

**Why this matters:** CJ's clothing is physically layered. A jacket sits on top of a shirt. If
both the `torso` (jacket) and `tshirt` (undershirt) slots were rendered simultaneously, the shirt
geometry would poke through the jacket. The `HIDE tshirt torso` rule prevents this: whenever a
torso-slot model is active, the tshirt layer is automatically hidden.

The 12 `HIDE` rules cover the principal overlap pairs:
- Jacket over shirt
- Long trousers over boxers/underwear
- Hat over bare-head
- Glasses over bare-face

This is why SA's clothing system is a *language* rather than a table: the `HIDE` rules encode the
spatial relationship between clothing layers, which cannot be expressed as a simple data column.

### 1.3 `IGNORE` / `ENDIGNORE` — block suspension (11/11 pairs)

```
IGNORE
   ... rules inside this block are inactive ...
ENDIGNORE
```

`IGNORE`/`ENDIGNORE` suspends all rule processing within the block. These are used for conditional
or disabled outfit groups — rules that exist in the file but are not active in the shipped game
(possibly development/debug outfits, or platform-specific items disabled on PC). The 11 pairs
document the number of such disabled blocks.

### 1.4 `CUTS` — cutscene overrides (20 instances)

```
CUTS  <bodypart>  <slot>  <model-name>  <texture-name>
```

`CUTS` is syntactically identical to `SETC` but applies only during **cutscene playback**. When a
cutscene is active, the cutscene-specific `CUTS` assignments override the player's chosen outfit.
This is how mission cutscenes can show CJ in a specific plot-required outfit (e.g., a tuxedo for
the Mafia mission set) regardless of what the player has equipped. The 20 `CUTS` rules map to
specific narrative moments requiring outfit control.

The relationship to `txdcut.ide` ([C48.1](../C48-Data-Folder-Sweep/01-load-and-boot-config.md)):
`txdcut.ide` assigns TXD parents for cutscene texture dictionaries, ensuring that cutscene-specific
clothing textures can inherit from their gameplay counterparts to save memory.

### 1.5 `EXCLUSIVE` / `ENDEXCLUSIVE` — mutual exclusion (3/3 pairs)

```
EXCLUSIVE
   ... group of mutually-exclusive items ...
ENDEXCLUSIVE
```

Items inside an `EXCLUSIVE` block cannot be worn simultaneously — selecting one deselects all
others in the block. Used for item categories where only one of several competing options makes
logical sense (e.g., a player can wear only one pair of shoes — wearing boots excludes sandals).
The 3 pairs define the three such exclusion groups in the shipped data.

---

## 2. Body-part slots: the attachment framework

The clothing system maps to named **body-part slots** that correspond to attachment frames in
CJ's RW clump hierarchy ([C50.1](../C50-RenderWare-Reference/01-the-object-model.md)):

| Slot name | Body region | DFF attachment point |
|---|---|---|
| `torso` | Torso / upper body | `Torso` frame |
| `tshirt` | Undershirt layer | `Torso` frame (hidden by HIDE rule) |
| `legs` | Trousers / lower body | `Pelvis` frame |
| `shoes` | Footwear | `Foot_L` / `Foot_R` frames |
| `hair` | Hair | `Head` frame |
| `head` | Head model (face, beards) | `Head` frame |
| `hat` | Hat / headwear | `Head` frame |
| `glasses` | Eyewear | `Head` frame |
| `watch` | Wrist accessory | `L_Hand` frame |
| `extra1/2` | Miscellaneous accessories | various |

Each slot can have **at most one** active model at a time. The slot system is an addressable
namespace over the attachment points in the RpClump frame hierarchy. `CClothesBuilder` (🟡)
walks the active slot assignments and uses `RpClumpAddAtomic` (or equivalent) to attach each
active model's `RpAtomic` to the appropriate `RwFrame`.

---

## 3. The `CClothesBuilder` assembly pipeline (🟡)

The runtime assembly class `CClothesBuilder` (gta-reversed naming, 🟡) is invoked when CJ's
outfit changes — at shop exit, at mission start with a `CUTS` override, or at game load with the
saved outfit state. Its pipeline:

```
1. Start with the base body clump (the naked CJ model — skin and body geometry)

2. For each body-part slot (torso → tshirt → legs → shoes → hair → head → hat → glasses → watch):
   a. Check if the slot has an active SETC assignment (the player's current equipped item)
   b. If yes: load the DFF for the model-name, look up the TXD for the texture-name,
              attach the RpAtomic to the slot's RwFrame in the base clump
   c. If no:  attach the default base-skin geometry for that slot

3. Evaluate all HIDE rules:
   a. For each HIDE rule: if the trigger slot is active, detach the hidden slot's RpAtomic
      from the clump (do not render it)

4. Evaluate all EXCLUSIVE rules:
   a. Ensure at most one item in each exclusive group is attached

5. Apply fat/muscle body scale (C55.3) to the resulting clump's frame hierarchy

6. The completed RpClump is the CJ drawable — used by the render system (C40)
   until the next outfit change
```

This pipeline runs on a **rebuild trigger**, not every frame. Between outfit changes, CJ's clump
is the cached output of the last rebuild. The rebuild is the expensive operation; the per-frame
render just submits the cached clump.

### 3.1 Model and texture loading

Each `SETC` rule references a DFF by basename and a TXD by texture name. The DFF is fetched from
the streaming system ([C1](../C1-Streaming/C1-Streaming.md)/[C2](../C2-CStreaming/C2-CStreaming.md))
by the model name. The TXD lookup is by name through the active texture dictionary stack
([C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md)). If the DFF is not yet streamed
in, `CClothesBuilder` must request it and wait — this is the brief load pause the player sees
when equipping new clothing.

### 3.2 The `CUTS` override during cutscenes

When the cutscene system ([C51.2](../C51-Executable-Lifecycle/02-frame-loop-and-shutdown.md)'s
`CCutsceneMgr::Update`) transitions to a cutscene, it calls the clothing system with the `CUTS`
rules in place of the player's `SETC` assignments. The same `CClothesBuilder` pipeline runs, but
steps 2a/2b now read from the `CUTS` table rather than the player's equipped slots. On cutscene
exit, the player's outfit is restored.

This mechanism is why mission-specific outfits in cutscenes are seamless: there is no separate
"CJ cutscene model" — it is the same system with a different data table.

---

## 4. The seven clothing shops in `shopping.dat`

`shopping.dat` partitions the clothing items by shop. Each shop is a sub-section inside the
`Clothes` section:

| Shop name | Price tier | Unlock condition |
|---|---|---|
| **BINCO** | Cheapest | Accessible from start |
| **Victim** | Low-mid | After moving to Idlewood |
| **eSé** | Mid | After Los Santos |
| **ProLaps** | Mid | After Los Santos |
| **Suburban** | Mid-high | After San Fierro missions |
| **ZIP** | High | After Las Venturas |
| **Ponsonbys** | Most expensive | Requires `sexy` stat above threshold |

Each item's line in `shopping.dat` is:
```
<internal-name>  <GXT-key>  respect <n>  sexy <n>  <price>
```

The `respect` and `sexy` thresholds are the **RPG gating mechanism**: Ponsonbys' premium suits
require a high `sexy` stat, which is earned by wearing expensive clothes and visiting the gym. The
stat system feeds back into itself — earning respect/sexy unlocks better clothes, which grants more
respect/sexy. This is the RPG progression loop encoded in `shopping.dat`.

---

## 5. Modding the clothing system

### 5.1 Adding a new outfit

An outfit mod adds entries to `clothes.dat` (new `SETC` rules), places new DFF/TXD files in the
appropriate locations (gta3.img or a loose file), and optionally adds the outfit to `shopping.dat`
for in-game purchase. The `SETC` → slot assignment → `CClothesBuilder` rebuild pipeline handles
the rest automatically — no code change is needed for new clothing that fits into existing slots.

### 5.2 Clipping issues

If new clothing clips through existing geometry, adding a `HIDE` rule in `clothes.dat` suppresses
the interfering slot. The `HIDE` verb is the clean solution; trying to fix clipping by moving
vertices in the DFF is brittle.

### 5.3 The `CUTS` table for scripted outfits

A CLEO or ASI mod that needs to force a specific outfit for a scripted mission can write directly
to the active `SETC` slot assignments (bypassing the shop UI) and trigger a `CClothesBuilder`
rebuild. Alternatively, adding a `CUTS` rule achieves the same effect during cutscene-flagged
sequences. ⏳ The exact function that triggers the rebuild and its calling convention are not
yet traced to a specific disassembly VA.

---

### Key takeaways

- `clothes.dat` is a **rule language**: `SETC` assigns body-part slot models (87 rules); `HIDE`
  suppresses overlapping layers (12 rules); `EXCLUSIVE` enforces mutual exclusion (3 pairs);
  `CUTS` overrides outfits during cutscenes (20 rules).
- The body-part slot system maps names (`torso`, `legs`, `hair`, etc.) to specific `RwFrame`
  attachment points in CJ's base `RpClump`.
- `CClothesBuilder` (🟡) evaluates the active rules on each outfit-change trigger and reassembles
  the full clump — the **result is cached** until the next change, so only rebuilds incur cost.
- The `CUTS` table gives the cutscene system outfit control without a separate CJ model — same
  pipeline, different data table.
- Shop prices and `respect`/`sexy` gating are all in `shopping.dat` — **the wardrobe is an RPG
  progression layer**, not just a cosmetic system.

**Continue:** [C55.2 — Haircuts, tattoos, and accessories →](02-hair-tattoos-accessories.md)
