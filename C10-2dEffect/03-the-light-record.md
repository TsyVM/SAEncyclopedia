# C10.3 — The Light Record

> **The one-sentence version:** eighty bytes carrying a colour, four ranges, five flag bytes and two
> 24-character texture names — and every light in San Andreas names the same two textures, a fact that
> a one-byte offset error nearly hid.

[← C10.2 — The effect-type census](02-effect-type-census.md) · [Chapter 10 hub](C10-2dEffect.md)

**Confidence:** ✅ Verified (layout to `+0x49`) / ⏳ (final 7 bytes)

---

## 1. The layout

Type 0, `dataSize` 80, 1,038 instances in `gta3.img`.

```c
struct TwoDEffectLight {          // 80 bytes
    uint8_t  colour[4];           // +0x00  RGBA
    float    coronaFarClip;       // +0x04
    float    pointlightRange;     // +0x08
    float    coronaSize;          // +0x0C
    float    shadowSize;          // +0x10
    uint8_t  flags[5];            // +0x14  five single-byte fields
    char     coronaTexName[24];   // +0x19
    char     shadowTexName[24];   // +0x31
    uint8_t  _undecoded[7];       // +0x49
};                                // = 0x50 = 80
```

✅ *Verified* through `+0x49`. The final 7 bytes are ⏳ open — they exist and are accounted for in the
size, but this pass did not determine their meaning.

A worked example, from `cj_bag_reclaim.dff`:

```
position        (24.9, 11.7, 2.1)     model-local
colour          (223, 206, 148, 100)  warm white, alpha 100
coronaFarClip   100.0
pointlightRange 18.0
coronaSize      0.80
shadowSize      8.00
coronaTexName   "coronastar"
shadowTexName   "shad_exp"
```

## 2. The off-by-one that nearly hid the result

The first parse placed the name fields at `+0x18` and produced this:

```
'@coronastar'  626      '`coronastar'  112      'Bcoronastar'   82
' coronastar'  128      'Dcoronastar'   42      'Acoronastar'   33
```

Eleven "distinct" corona textures, each a recognisable string with one junk byte in front.

That leading byte is the fifth flag field. Moving the names to `+0x19`:

| Name offset | Lights | Clean names | Distinct corona names |
|---|---:|---:|---:|
| `+0x18` | 1,038 | 171 | 11 |
| **`+0x19`** | **1,038** | **1,038** | **1** |

✅ At `+0x19`, **all 1,038 lights** yield `coronastar` and `shadowTexName` yields `shad_exp` — a single
value each, no exceptions.

**The check that caught it:** a fixed-width name field should produce a small set of clean strings. Eleven
variants that differ only in their first character is not a naming convention — it is a misaligned read.
Counting *distinct values* is a cheap, general test for string-field alignment, and it is worth applying
to any fixed-width name field before trusting it.

The same class of error as the `peds.col` walk ([C6.2 §4](../C6-Collision/02-header-and-bounds.md)):
plausible-looking output that parses without complaint.

## 3. One corona, one shadow, in the whole game

```
coronaTexName : "coronastar"   × 1,038 / 1,038
shadowTexName : "shad_exp"     × 1,038 / 1,038
```

✅ Verified. Every placed light in `gta3.img` references the same two textures.

The variation between lights is entirely in **colour, size and range** — the four floats and the RGBA.
San Andreas's night-time look is one corona sprite and one shadow blob, tinted and scaled a thousand
different ways.

**Consequences.** Replacing `coronastar` changes every light in the game at once — the single
highest-leverage texture edit available. And a lighting mod that adds new corona textures is adding
something the shipped content never used, so any engine behaviour around per-light textures is
effectively untested by the retail data set.

## 4. What the fields do

🟡 *Reasoned* — consistent with the values but not confirmed against rendering code:

- **`coronaFarClip`** — 100.0 in the sample: the distance past which the corona stops drawing.
- **`pointlightRange`** — 18.0: the radius over which the light affects nearby geometry, distinct from
  the corona's visibility.
- **`coronaSize`** / **`shadowSize`** — 0.80 and 8.00: the sprite scales. That the shadow is ten times
  the corona is the usual relationship — a small bright point casting a broad soft pool.
- **`colour`** — `(223, 206, 148, 100)` is a warm incandescent white at 39 % alpha.

The two ranges being *separate* fields is the interesting design detail: a light can be visible from far
away while illuminating only its immediate surroundings, which is exactly what a street lamp needs.

⏳ **Open:** the five flag bytes at `+0x14` and the seven trailing bytes at `+0x49`. Community sources
name several of these (corona show mode, flare type, shadow intensity, Z distance), but per
[C10.1 §3](01-the-record-and-corrections.md) this chapter does not adopt names it has not derived —
that is precisely the mistake that produced the `0x253F2FE` correction.

Deriving them is straightforward and was not done here: census each byte's value distribution across
1,038 records and correlate against the models that carry them.

## 5. Where the lights are

1,038 lights across 12,955 models. Combined with
[C10.2 §4](02-effect-type-census.md)'s clustering, lights concentrate in the models that have any
effects at all — the `cj_bag_reclaim` example carries several at once, spaced along an axis at regular
intervals (`x = 24.9, 11.0, −0.7` with identical colour and size), which is a strip of ceiling lights
authored as one model.

Positions are **model-local**. A light's world position is its offset transformed by the entity's
placement, which comes from the IPL — so the same lit model placed twenty times produces twenty
lighting rigs from one 80-byte record. That is the efficiency argument for putting lights on geometry
rather than in a world file.

---

### Key takeaways

- The light record is **80 bytes**: RGBA, four floats, five flag bytes, two 24-char names, seven
  undecoded trailing bytes.
- ⚠️ The names sit at **`+0x19`, not `+0x18`** — a one-byte error produced eleven plausible-looking
  "distinct" texture names.
- **Counting distinct values is a general alignment test** for fixed-width string fields: 11 variants
  differing only in the first character means a misread, not a convention.
- **Every light in the game uses `coronastar` and `shad_exp`** — 1,038 of 1,038. Variation is entirely
  colour, size and range.
- Replacing `coronastar` restyles **every light in San Andreas** at once.
- `coronaFarClip` and `pointlightRange` are **separate** — visible far, illuminating near.
- Positions are **model-local**, so one record lights every placement of its model.
- ⏳ Five flag bytes and seven trailing bytes remain underived — and are deliberately **not** filled in
  from community names.

**Continue:** [Chapter 10 hub](C10-2dEffect.md) · next chapter: `C11 — IDE & IPL: the World's Data Model`
