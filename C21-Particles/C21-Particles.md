# Chapter 21 — Particles: the `effects.fxp` Effect Library

> **Goal of this chapter:** decode `models/effects.fxp` — the definition of every particle system in the
> game. The file turns out **not to be binary at all**: it is 616,708 bytes of plain CRLF text. So the
> proof standard shifts from record widths to **declared counts closing**, and it closes completely — a
> parser driven only by the file's own `NUM_*` fields consumes **all 43,617 lines with nothing left
> over**, and all **29** effect-info types have a single constant schema.

**Subsystem category:** Rendering / effects
**Depends on:** [C0.1 — The HOODLUM layer](../C0-Binary-Identity/01-the-hoodlum-layer.md) (address
resolution into the packed executable)
**Ties:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [C38](../C38-Skybox-And-Clouds/C38-Skybox-And-Clouds.md), [C40](../C40-Render-Pipeline/C40-Render-Pipeline.md), [C44](../C44-Shaders/C44-Shaders.md)
**RE status:** Verified
**Confidence:** ✅ Verified (encoding, grammar, exact closure, all 29 schemas, the executable's token set)
/ 🟡 (the meaning of the curve time axis) / ⏳ (the version word `109`, three numeric enums)

---

## Deep-dive pages

- [C21.1 — The grammar](01-the-grammar.md): a five-level nested text format whose only variable-length
  constructs are three counted lists — and the count-driven parser that proves it by consuming the file
  exactly.
- [C21.2 — The effect library](02-the-effect-library.md): what the 82 systems, 161 emitters, 1,470 info
  blocks and 6,563 keyframes actually contain — textures, blend modes, LOD ranges, and the exceptions
  worth naming.
- [C21.3 — The parser in the executable](03-the-parser-in-the-executable.md): 34 block tags compiled into
  `gta_sa.exe`, **three of which the shipped file never uses** — and the proof that scalar fields are read
  by position, not by name.

---

## 21.1 The result first

| Claim | Evidence |
|---|---|
| The file is **plain text**, CRLF | 616,708 bytes, 43,617 lines, **zero** bytes outside printable ASCII + CRLF ✅ |
| **82 systems, 161 emitters, 1,470 info blocks, 6,563 keyframes** | a count-driven parse ✅ |
| The grammar is **complete** | the parser consumes **43,617 / 43,617** lines, **0 left over** ✅ |
| `NUM_PRIMS` totals the emitters | Σ = **161** = emitter blocks found ✅ |
| `NUM_INFOS` totals the info blocks | Σ = **1,470** = info blocks found ✅ |
| `NUM_KEYS` totals the keyframes | Σ = **6,563** = keyframe blocks found ✅ |
| All **29** info types have one fixed schema | no type varies across its instances ✅ |
| Every info type is a literal in the exe | **29 / 29** ✅ |
| The exe supports **3 types the file never uses** | `ATTRACTLINE`, `COLOURRANGE`, `SMOKE` ✅ |
| Curve times are ordered | **4,120 / 4,120** curves have ascending `TIME` ✅ |

**Every line of the file is accounted for by the grammar, and the grammar's block vocabulary matches the
executable's exactly — with three tokens to spare.**

## 21.2 The surprise: it is not a binary format

The handoff listed `effects.fxp` as "a single self-contained record-array binary" — a reasonable guess
from the extension and the size. The first sixteen bytes settle it otherwise:

```
00000000:  46 58 5f 50 52 4f 4a 45 43 54 5f 44 41 54 41 3a   FX_PROJECT_DATA:
```

There is no header, no count, no offset table. The whole file is `KEY: value` lines and `FX_*_DATA:` block
tags separated by CRLF, and it contains **no byte outside printable ASCII**. The retail PC build ships the
particle library as the exporter's text output.

That changes the method but not the standard. In a binary format the encyclopedia proves a layout by
showing `count × width` leaves no residue. In a text format the equivalent is that **a parser which knows
only the grammar and the declared counts must consume the file exactly** — no line unread, no count
overrunning its block. That is a strictly stronger test than "the format looks plausible", because a wrong
guess about where any block ends desynchronises the parse and it cannot land on the final
`FX_PROJECT_DATA_END:` with an empty remainder.

## 21.3 The shape of an effect

Five levels, each contained by the one above:

```
FX_PROJECT_DATA:                              the file
└── FX_SYSTEM_DATA:            × 82           one named effect  (prt_blood, fire, camflash …)
    │   version 109, FILENAME, NAME, LENGTH, PLAYMODE, CULLDIST, BOUNDINGSPHERE
    └── FX_PRIM_EMITTER_DATA:  × 161          one emitter — the thing that spawns particles
        │   FX_PRIM_BASE_DATA: NAME, MATRIX, TEXTURE ×4, ALPHAON, SRCBLENDID, DSTBLENDID
        │   … LODSTART, LODEND
        └── FX_INFO_<TYPE>_DATA:  × 1,470     one behaviour: size, colour, force, wind, friction …
            └── <curve name>:     × 4,120     e.g. SIZEX, RED, FORCEZ
                └── FX_INTERP_DATA:           LOOPED, NUM_KEYS
                    └── FX_KEYFLOAT_DATA: × 6,563    TIME, VAL
```

The design is worth a sentence, because it explains why the file is so uniform. **Nothing in a particle
system is a constant** — every quantity an artist can set (a size, a colour channel, an emission rate, a
force component) is a *curve*: a list of `{TIME, VAL}` keyframes with a `LOOPED` flag. An effect that
never changes size still stores its size as a one-key curve, which is exactly why 2,721 of the 4,120
curves have `NUM_KEYS: 1`. The format has one idea and applies it everywhere, and the 29 info types are
just names for which curves travel together.

## 21.4 What this chapter does not claim

⏳ The **version word `109`** on each `FX_SYSTEM_DATA:` block is a bare number on its own line, constant
across all 82 systems. Position certain; meaning not derived. It is *not* a count — there are 82 systems,
not 109.

⏳ Three fields are small integers whose enumerations are not decoded: `PLAYMODE` (0 on 27 systems, 1 on
7, 2 on 48), and `SRCBLENDID` / `DSTBLENDID`. The blend IDs are 🟡 *reasoned* to be Direct3D
`D3DBLEND` values — `SRCBLENDID` is **4 on all 161** emitters and `DSTBLENDID` is 5 or 1, which matches
`D3DBLEND_SRCALPHA` / `INVSRCALPHA` / `ONE` and reproduces the standard alpha-blend and additive pairs a
particle system needs — but nothing in the shipped data forces the mapping, so it is not promoted.

🟡 **`TIME` is a normalised position along the effect's `LENGTH`.** 4,103 of 4,120 curves start at exactly
`TIME: 0.000` and the overwhelming majority end at `1.000`. But 17 curves carry keys beyond 1.0 — up to
7.5 — so the normalisation is a convention the exporter usually follows rather than a bound the format
enforces. The exceptions are named in [C21.2](02-the-effect-library.md).

---

### Key takeaways

- `effects.fxp` is **plain CRLF text**, not a binary record array — 616,708 bytes, 43,617 lines, no byte
  outside printable ASCII.
- ✅ The grammar is **complete and closes**: a parser driven only by `NUM_PRIMS` / `NUM_INFOS` /
  `NUM_KEYS` consumes **43,617 of 43,617** lines with **nothing left over**.
- ✅ **82 systems → 161 emitters → 1,470 info blocks → 4,120 curves → 6,563 keyframes**, each level's
  declared total matching the blocks actually found.
- ✅ All **29** effect-info types have a **single constant schema**; none varies across its instances.
- ✅ The executable contains **34** `FX_*_DATA` tokens — the 29 used types, the two container types, and
  **three the shipped file never uses**.
- 🧭 Every artist-settable quantity is a **curve**, never a constant; 2,721 curves hold a single key.
- ⏳ The version word `109`, `PLAYMODE`, and the blend-ID enumerations are recorded as open.

**Continue:** [C21.1 — The grammar](01-the-grammar.md)

## See also (forward links)

The particle systems here are submitted to the frame by [C40 — Render Pipeline](../C40-Render-Pipeline/C40-Render-Pipeline.md) (effects stage), share the immediate-mode path with [C38 — Skybox & Clouds](../C38-Skybox-And-Clouds/C38-Skybox-And-Clouds.md) (coronas, sun, moon), and run through the fixed-function im3d pipeline documented in [C44 — Shaders](../C44-Shaders/C44-Shaders.md).

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [C38](../C38-Skybox-And-Clouds/C38-Skybox-And-Clouds.md), [C40](../C40-Render-Pipeline/C40-Render-Pipeline.md), [C44](../C44-Shaders/C44-Shaders.md)
- **Known bugs / gotchas:** 3 exe-declared particle types are never used; LODSTART/END belong to the emitter (parse trap).
- **Modding:** effects.fxp is the particle-mod file; submitted via C40 im3d (C44).
- **Performance:** 43,617 lines parsed at load; per-emitter draw at runtime.
