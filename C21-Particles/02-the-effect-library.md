# C21.2 — The Effect Library

> **The one-sentence version:** 82 named effects built from 161 emitters and 1,470 behaviour blocks drawn
> from 29 types, in which every artist-settable quantity — size, colour, force, emission rate — is stored
> as a keyframed curve rather than a constant, and 2,721 of those 4,120 curves hold exactly one key.

[← Chapter 21 hub](C21-Particles.md) · [Prev: C21.1 — The grammar](01-the-grammar.md) ·
[Next: C21.3 — The parser in the executable →](03-the-parser-in-the-executable.md)

**Confidence:** ✅ Verified (counts, all 29 schemas, curve ordering, field distributions) / 🟡 (blend IDs,
the time axis) / ⏳ (`PLAYMODE`, `TIMEMODEPRT`)

---

## 1. Eighty-two effects, and where they came from

Each system carries a `FILENAME` field holding the **absolute path on the build machine** it was exported
from. Those paths survive in the retail file and partition the library by intended caller:

| Source directory | Systems | What lives there |
|---|---:|---|
| `…\gta_pc\systems\code` | 37 | effects spawned by engine code |
| `…\gta_pc\particles` | 17 | the `prt_*` primitives |
| `…\gta_pc\systems\script` | 15 | effects a mission script triggers |
| `…\gta_pc\systems\map` | 11 | placed effects (`WS_factorysmoke` and friends) |
| `…\gta_pc\systems\object` | 2 | object-attached effects |

All 82 sit under `X:\SA\FxTools\Data\effects\gta_pc\`, so the whole library was exported from one tool
against one tree. The `gta_pc` component is the only platform token present — this file is the PC build's
library and contains no trace of a console variant.

✅ *Verified:* 82 systems, names ranging from the primitives (`prt_blood`, `prt_smoke_huge`, `prt_spark`)
to composed effects (`fire_car`, `camflash`, `jetpack`, `nitro`, `teargasAD`).

Most effects are simple: **41 of 82 have a single emitter**, 18 have two, and the largest — two systems —
have six.

## 2. The emitter

All 161 emitters are of one type, `FX_PRIM_EMITTER_DATA`, and each opens with a ten-line
`FX_PRIM_BASE_DATA` block. The distributions across all 161 are informative in themselves:

| Field | Distribution |
|---|---|
| `MATRIX` | identity rotation, zero translation on **151 / 161**; 10 emitters are transformed |
| `TEXTURE` | 35 distinct; `bullethitsmoke` (38), `sphere` (21), `smokeII_3` (11), `sphere_CJ` (9) |
| `TEXTURE2` | used **once** in the entire file; `TEXTURE3` and `TEXTURE4` **never** |
| `ALPHAON` | `1` on **all 161** |
| `SRCBLENDID` | `4` on **all 161** |
| `DSTBLENDID` | `5` on 103, `1` on 58 |

> The single emitter that uses a second texture is **`prt_splash`**, whose `ParticleEmitter` sets
> `TEXTURE2: splash_up2`. Three of the four texture slots exist in the format and are used zero times;
> the fourth is used once. Named rather than rounded to "textures are unused".

🟡 *Reasoned:* `SRCBLENDID` / `DSTBLENDID` are Direct3D `D3DBLEND` enumerants. `SRCBLENDID = 4` is
`D3DBLEND_SRCALPHA`; `DSTBLENDID` 5 and 1 are `INVSRCALPHA` and `ONE`, which give exactly the two blend
modes a particle system needs — ordinary alpha compositing (103 emitters) and additive glow (58, which is
where the fire, spark and light effects land). The mapping is consistent and complete, but nothing in the
shipped data *forces* it, so it stays 🟡. ⏳ `PLAYMODE` (`2` on 48 systems, `0` on 27, `1` on 7) has no
such external anchor and is left uninterpreted.

### 2.1 LOD, and thirteen emitters that disable it

Each emitter ends with `LODSTART` and `LODEND`, a distance range spanning 5.0 to 300.0 across the library.
On **148 of 161** emitters `LODSTART < LODEND`, as a fade band requires.

⚠️ On the other **13**, `LODSTART == LODEND` exactly — a zero-width band. They occur in ten systems:
`camflash`, `fire`, `fire_bike`, `fire_car`, `fire_large`, `fire_med`, `flamethrower`, `jetpack`, `nitro`
and `shootlight` (which accounts for four of the thirteen, all at `300.000 / 300.000`).

🟡 The reading is that a zero-width band means "pop, don't fade" — and the membership of that list is
suggestive, since every one of them is a bright, short-lived light or flame where a fade would be more
visible than a pop. ⏳ Not proven; recorded as a real and named property of thirteen emitters.

## 3. Twenty-nine behaviours

The 1,470 info blocks fall into 29 types, each with a fixed schema of scalar fields (`*`) and named curves:

| Type | n | Children |
|---|---:|---|
| `FX_INFO_SIZE_DATA` | 161 | `TIMEMODEPRT*` `SIZEX` `SIZEY` `SIZEXBIAS` `SIZEYBIAS` |
| `FX_INFO_EMLIFE_DATA` | 147 | `LIFE` `BIAS` |
| `FX_INFO_EMRATE_DATA` | 144 | `RATE` |
| `FX_INFO_EMSPEED_DATA` | 140 | `SPEED` `BIAS` |
| `FX_INFO_EMANGLE_DATA` | 122 | `MIN` `MAX` |
| `FX_INFO_FORCE_DATA` | 111 | `TIMEMODEPRT*` `FORCEX` `FORCEY` `FORCEZ` |
| `FX_INFO_ROTSPEED_DATA` | 108 | `TIMEMODEPRT*` `MINCW` `MAXCW` `MINCCW` `MAXCCW` |
| `FX_INFO_COLOURBRIGHT_DATA` | 102 | `TIMEMODEPRT*` `RED` `GREEN` `BLUE` `ALPHA` `BIAS` |
| `FX_INFO_EMROTATION_DATA` | 85 | `ANGLEMIN` `ANGLEMAX` |
| `FX_INFO_EMDIR_DATA` | 65 | `DIRX` `DIRY` `DIRZ` |
| `FX_INFO_EMSIZE_DATA` | 62 | `RADIUS` `SIZEMINX` `SIZEMAXX` `SIZEMINY` `SIZEMAXY` `SIZEMINZ` `SIZEMAXZ` |
| `FX_INFO_COLOUR_DATA` | 48 | `TIMEMODEPRT*` `RED` `GREEN` `BLUE` `ALPHA` |
| `FX_INFO_FRICTION_DATA` | 46 | `TIMEMODEPRT*` `FRICTION` |
| `FX_INFO_WIND_DATA` | 38 | `TIMEMODEPRT*` `WINDFACTOR` |
| `FX_INFO_SELFLIT_DATA` | 28 | `TIMEMODEPRT*` |
| `FX_INFO_DIR_DATA` | 12 | `TIMEMODEPRT*` `X` `Y` `Z` |
| `FX_INFO_HEATHAZE_DATA` | 9 | `TIMEMODEPRT*` |
| `FX_INFO_SPRITERECT_DATA` | 8 | `TIMEMODEPRT*` `TOP` `BOTTOM` `LEFT` `RIGHT` |
| `FX_INFO_JITTER_DATA` | 8 | `TIMEMODEPRT*` `JITTERFACTOR` |
| `FX_INFO_FLAT_DATA` | 6 | `TIMEMODEPRT*` `RX` `RY` `RZ` `UX` `UY` `UZ` `AX` `AY` `AZ` |
| `FX_INFO_EMPOS_DATA` | 5 | `X` `Y` `Z` |
| `FX_INFO_FLOAT_DATA` | 3 | `TIMEMODEPRT*` |
| `FX_INFO_NOISE_DATA` | 3 | `TIMEMODEPRT*` `NOISE` |
| `FX_INFO_UNDERWATER_DATA` | 3 | `TIMEMODEPRT*` |
| `FX_INFO_TRAIL_DATA` | 2 | `TIMEMODEPRT*` `TRAILTIME` `SCREENSPACE` |
| `FX_INFO_GROUNDCOLLIDE_DATA` | 1 | `TIMEMODEPRT*` `BOUNCE` `SPEEDMULT` `BOUNCEERROR` |
| `FX_INFO_ANIMTEX_DATA` | 1 | `TIMEMODEPRT*` `TEXID` |
| `FX_INFO_EMWEATHER_DATA` | 1 | `WINDMIN` `WINDMAX` `RAINMIN` `RAINMAX` |
| `FX_INFO_ATTRACTPT_DATA` | 1 | `TIMEMODEPRT*` `POSX` `POSY` `POSZ` `FORCE` |

✅ *Verified:* every one of these schemas is **constant across all instances of its type** — that is the
independent check on the grammar's one inferred rule ([C21.1 §4](01-the-grammar.md)).

Two structural observations fall straight out of the table. First, `FX_INFO_SIZE_DATA` appears **161
times** — exactly once per emitter. It is the only mandatory behaviour; every emitter has a size, and
nothing else is universal. Second, the `EM`-prefixed types (`EMLIFE`, `EMRATE`, `EMSPEED`, `EMANGLE`,
`EMROTATION`, `EMDIR`, `EMSIZE`, `EMPOS`, `EMWEATHER`) describe the **emitter's** behaviour while the rest
describe the **particle's**, and only the latter carry a `TIMEMODEPRT` scalar — every non-`EM` type has it
and no `EM` type does, without exception across 1,470 blocks.

⏳ `TIMEMODEPRT` is that scalar, present on 699 blocks. 🟡 The perfect correlation with per-particle types
says it selects what the curve's time axis is measured against — effect lifetime versus particle lifetime
— which is precisely the distinction a per-particle behaviour needs and a per-emitter one does not. The
name supports it. The bytes do not prove it, so it stays 🟡.

## 4. Curves and keyframes

4,120 curves hold 6,563 keyframes between them, and the distribution is lopsided:

| `NUM_KEYS` | Curves |
|---:|---:|
| 1 | 2,721 |
| 2 | 690 |
| 3 | 483 |
| 4 | 182 |
| 5 | 39 |
| 6 | 1 |
| 21 | 4 |

(`1×2,721 + 2×690 + 3×483 + 4×182 + 5×39 + 6×1 + 21×4 = 6,563` — the distribution accounts for every
keyframe. The single six-key curve is an `FX_INFO_EMRATE_DATA` rate; the four twenty-one-key curves are
the longest in the game.)

Two thirds of every curve in the game is a single point — a constant dressed as an animation. That is the
cost of the format's one idea, and it is why a 616 KB text file describes only 82 effects.

The keyframes are well-formed under every test applied:

- ✅ **4,120 / 4,120** curves have `TIME` values in ascending order.
- ✅ **4,103 / 4,120** curves begin at exactly `TIME: 0.000`.
- `LOOPED` is `0` on 4,116 curves and `1` on **4**.

⚠️ **24 keyframes across 17 curves carry `TIME > 1.0`**, up to 7.5 — so the time axis is not bounded at
1.0 by the format. They fall in exactly five systems, and they are listed rather than smoothed away:

| System | Curves affected | Max `TIME` |
|---|---|---:|
| `WS_factorysmoke` | `DIRX` `DIRY` `DIRZ` `WINDFACTOR` `SIZEX` `SIZEY` `SIZEXBIAS` `SIZEYBIAS` | **7.5** |
| `water_fnt_tme` | `RATE` (×2) | 5.0 |
| `water_splash` | `SIZEX` `SIZEY` `SIZEXBIAS` `SIZEYBIAS` | 1.5 |
| `teargasAD` | `RATE` | 2.0 |
| `exhale` | `SPEED` `BIAS` | 1.5 |

The 17 curves that do not start at `TIME: 0.000` are concentrated the same way — 6 in `EMLIFE`, 4 in
`SIZE`, 4 in `EMSPEED`, 3 in `EMRATE`.

🟡 The working reading is that `TIME` is normalised against the system's `LENGTH`, which the exporter
respects almost everywhere; the outliers are long-running ambient effects where a curve deliberately runs
past one nominal cycle. ⏳ Not proven.

---

### Key takeaways

- ✅ **82 systems / 161 emitters / 1,470 info blocks / 4,120 curves / 6,563 keyframes**, exported from one
  build tree (`X:\SA\FxTools\Data\effects\gta_pc\`) and partitioned by caller — code, script, map, object.
- ✅ All **29** info-type schemas are **constant across every instance**.
- 🧭 `FX_INFO_SIZE_DATA` appears **exactly once per emitter** — the only mandatory behaviour.
- 🧭 The `EM*` types describe the emitter and the rest describe the particle, and **only the latter carry
  `TIMEMODEPRT`** — a clean split with no exceptions in 1,470 blocks.
- ✅ Curves are well-formed: **4,120/4,120** ascending in `TIME`, **4,103/4,120** starting at 0.000.
- ⚠️ Named exceptions: **13 emitters** with `LODSTART == LODEND`; **24 keyframes** with `TIME > 1.0` in
  five systems; **one** use of `TEXTURE2` (`prt_splash`) and none of `TEXTURE3`/`TEXTURE4`.
- 🟡 Blend IDs read as `D3DBLEND` (alpha on 103 emitters, additive on 58); `TIMEMODEPRT` reads as the
  curve's time base. ⏳ `PLAYMODE` is left uninterpreted.

**Continue:** [C21.3 — The parser in the executable](03-the-parser-in-the-executable.md)
