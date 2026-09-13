# C55.2 — Haircuts, Tattoos, and Accessories

> **The one-sentence version:** hair, tattoos, and accessories are the three cosmetic layers
> above clothing — hair and accessories are mesh attaches to named `RwFrame` nodes in CJ's clump,
> tattoos are texture-compositing operations on the body-skin `RwRaster`, and all are purchased
> through sub-sections of `shopping.dat` with no stat gating.

**Subsystem category:** Gameplay — CJ cosmetic layers
**Depends on:** [C55.1](01-clothing-rule-language.md) (`CClothesBuilder` pipeline),
[C48.2](../C48-Data-Folder-Sweep/02-economy-and-customization.md) (`shopping.dat` structure),
[C9](../C9-Materials-And-Textures/C9-Materials-And-Textures.md) (texture lookup and TXD stack),
[C50](../C50-RenderWare-Reference/C50-RenderWare-Reference.md) (RW frame/atomic attachment)
**RE status:** Documented — `shopping.dat` sections ✅; runtime mechanism 🟡 (gta-reversed naming)
**Confidence:** ✅ for all `shopping.dat` section data · 🟡 for the runtime attach/composite
mechanisms (structurally confirmed, individual VAs not traced)

---

## 1. Hair: a mesh attached to the `Head` frame

Hair in SA is a **separate geometry mesh** (`RpAtomic`) attached to the `Head` `RwFrame` of CJ's
base clump at the time the hairstyle is applied. It is not baked into the base body model.

### 1.1 The attachment mechanism (🟡)

When CJ visits a barber and selects a style, the clothing/appearance system:

1. Detaches any currently-attached hair atomic from the `Head` frame
2. Streams in the DFF for the selected hair style (if not already resident)
3. Attaches the new hair's `RpAtomic` to the `Head` frame using `RpClumpAddAtomic` (or the
   equivalent frame-attach call — 🟡, not individually traced to a VA)
4. Triggers a clump rebuild if the hair change affects HIDE-rule evaluation (some hats hide
   bare-head hair; changing hair may re-evaluate hat visibility)

The `Head` frame is a named `RwFrame` in CJ's skeleton hierarchy. Its world transform is driven
by the head bone during animation — so whichever mesh is attached to it inherits the head's
rotations, making the hair visually "part of" CJ's head.

### 1.2 The `shopping.dat` haircut section

The `Haircuts` section in `shopping.dat` (🔷 approximate structure, confirmed pattern from
C48.2's `shopping.dat` documentation) lists each barber style:

```
section Haircuts
  Bald           HAIR_00  respect 0  sexy 0  0
  Caesar         HAIR_01  respect 0  sexy 0  150
  Cornrows       HAIR_02  respect 0  sexy 0  200
  ... (approximately 15 styles total)
end
```

Haircuts have **no stat gating** (`respect 0 sexy 0`) — any barber is accessible from the
beginning of the game and any style can be selected without a prerequisite. The price ranges from
free (shave-head) to a few hundred dollars for premium styles. Each `GXT-key` (`HAIR_00`,
`HAIR_01`, etc.) maps to the display name in the GXT table ([C19](../C19-GXT-Text/C19-GXT-Text.md)).

### 1.3 Hair and the `HIDE` rules

The clothing system's `HIDE` rules include hat/hair interactions: wearing a hat suppresses the
hair slot, because a hat mesh attached at the same `Head` frame would clip through hair geometry.
The `HIDE hair hat` rule (🟡 — exact verb syntax confirmed from the HIDE pattern established in
C55.1; this specific pair is inferred from gameplay behaviour) ensures only one occupies the head
at a time. The hat/glasses exception: some hats leave room for glasses, and those pairs do not
have a mutual HIDE.

---

## 2. Tattoos: texture compositing on the body skin

Tattoos work differently from hair — they are not separate mesh objects but **texture overlays
composited onto CJ's body skin texture** at the point of application.

### 2.1 The compositing operation (🟡)

When CJ gets a tattoo at a parlour:

1. The base body skin texture (the `RwRaster` used by the torso/limb geometry) is the target
2. The tattoo design — a texture with an alpha channel defining the tattoo boundary — is the source
3. The system blits the tattoo source onto the body skin raster at a fixed position (left arm,
   right arm, chest, back, etc.) using a texture copy with alpha masking

The result is a modified body-skin raster that permanently includes the tattoo pattern. This is
why tattoos are persistent without being a separate mesh — they are baked into the skin texture.
The alpha channel of the tattoo source controls blending: a sharp-edged tattoo uses a mostly
binary alpha; a softer or tribal design may use graduated alpha.

**Implication for modding**: replacing a tattoo design means replacing the tattoo source texture
in the TXD. The compositing writes into the skin raster in-memory; to remove a tattoo you would
need to restore the clean skin raster (which the game does by resetting to a pristine body texture
and re-applying only the currently active tattoos).

### 2.2 The `shopping.dat` tattoos section

The `Tattoos` section lists designs by location and parlour:

```
section Tattoos
  TattooParlour1_Chest_1    TATO_C1  respect 0  sexy 0  200
  TattooParlour1_LArm_1     TATO_L1  respect 0  sexy 0  150
  TattooParlour2_RArm_Tribal TATO_R3  respect 0  sexy 0  250
  ... (approximately 50 designs total across multiple parlours)
end
```

Like haircuts, tattoos have **no stat gating**. The parlour location is encoded in the section
name rather than gating. Each design name has a positional component: `_Chest_`, `_LArm_`,
`_RArm_`, `_Back_`, etc., indicating which body region the tattoo targets. This is used by the
compositing step to determine the blitting position on the skin texture.

### 2.3 Multiple tattoos: compositing order

CJ can wear multiple simultaneous tattoos across different body regions. Each tattoo is applied
independently to its target region of the body skin. The order of application matters when two
tattoos overlap the same region — the second tattoo is blitted on top of the first. In practice,
the parlour UI prevents same-region duplication, but the compositing system (🟡) is capable of
layering.

---

## 3. Accessories: mesh attaches to named frames

Accessories (hats, glasses, chains, watches) follow the same attachment mechanism as hair — they
are `RpAtomic` meshes attached to named `RwFrame` nodes — but use different frames and are treated
as separate slots in the `clothes.dat` system.

### 3.1 The accessory slots and their frames

| Slot | Frame | Typical item |
|---|---|---|
| `hat` | `Head` frame | Baseball cap, cowboy hat, beanie |
| `glasses` | `Head` frame (sub-frame, in front of `Hat`) | Sunglasses, tinted specs |
| `watch` | `L_Hand` frame | Wristwatch, bracelet |
| `extra1` | varies | Chain / necklace (torso-attach) |
| `extra2` | varies | Second accessory slot |

The `hat` and `glasses` slots share the `Head` frame. Rendering order is controlled by their
frame depth (glasses are a child frame of the hat frame, placing them in front). The game's
`HIDE hat glasses` and `HIDE glasses hat` logic is absent — hats and glasses can coexist if the
pair does not clip, but some hat models do clip eyewear. The visual result depends on the specific
DFF geometry.

### 3.2 The `shopping.dat` accessories sections

Accessories appear in dedicated sections of `shopping.dat`. The pattern follows the clothing
section format (name, GXT key, respect/sexy thresholds, price). Accessories similarly have minimal
stat gating compared to premium clothing — most are available from early in the game at relatively
low cost.

---

## 4. The persistence model: save file storage

CJ's active outfit state (which SETC slots are active, which hairstyle, which tattoos, which
accessories) is persisted in the **save file**. On game load, the clothing system:

1. Reads the saved slot assignments from the save file
2. Calls `CClothesBuilder` with those assignments to rebuild the active clump
3. Re-applies any active tattoos to the body skin raster

The save-file format for clothing is a set of item IDs, one per slot, with 0 indicating "default
base skin." The item IDs correspond to positions in the `shopping.dat` section tables.

**Implication for save-file mods**: editing CJ's appearance in a save editor requires understanding
the per-slot ID layout. An ID that is out of range (no corresponding `SETC` rule) will cause
`CClothesBuilder` to reference a non-existent model, typically resulting in a missing-slot visible
hole in the geometry. A robust appearance mod validates all slot IDs against the current
`clothes.dat` before writing.

---

## 5. The full cosmetic layer interaction

With all three cosmetic layers accounted for, the full appearance pipeline is:

```
Body geometry (fat/muscle-scaled base model — C55.3)
    + Clothing models (SETC rules → CClothesBuilder → RpAtomic attaches)
    + HIDE rules (suppress clipping layers)
    = Dressed body clump

Dressed body clump
    + Hair (RpAtomic attach to Head frame)
    + Hat/glasses (RpAtomic attach to Head frame)
    + Watch/accessories (RpAtomic attach to L_Hand, etc.)
    = Base appearance clump

Base appearance clump
    + Tattoo texture (composited onto body skin RwRaster at purchase time)
    = Final CJ drawable
```

The **dressed body clump** is rebuilt by `CClothesBuilder` on outfit change. The **cosmetic
attaches** (hair, accessories) are applied on top of the dressed clump. The **tattoo composite**
is applied to the base skin raster at tattoo-purchase time and persists in the raster until
another tattoo session resets it.

---

### Key takeaways

- **Hair and accessories** are `RpAtomic` mesh attaches to named `RwFrame` nodes (`Head`,
  `L_Hand`, etc.) — they inherit the bone's animation transform and are detached/reattached
  when the player changes styles.
- **Tattoos** are **texture composites** — the tattoo design is alpha-blitted onto the body skin
  `RwRaster` at the time of application and baked into the texture; they are not separate meshes.
- Both haircuts and tattoos have **no stat gating** in `shopping.dat` — unlike premium clothing,
  all styles are accessible regardless of respect/sexy level.
- All cosmetic state is saved per-slot in the save file; invalid slot IDs produce visible geometry
  holes, making ID-range validation critical for save-editing tools.

**Previous:** [C55.1 — The clothing rule language →](01-clothing-rule-language.md)
**Continue:** [C55.3 — Fat, muscle, and the body-stat RPG system →](03-body-stats-fat-muscle.md)
