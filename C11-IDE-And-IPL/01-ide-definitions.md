# C11.1 — IDE: the Definition Side

> **The one-sentence version:** nine section types across 59 files assign 14,266 model IDs, every one
> of them inside the 0–19999 range C3.1 derived from a `cmp` instruction — and seven of them are
> assigned twice.

[← Chapter 11 hub](C11-IDE-And-IPL.md) · [Next: C11.2 — IPL placements →](02-ipl-placements.md)

**Confidence:** ✅ Verified over all 59 files

---

## 1. The file model

An `.ide` is a plain-text file of named sections:

```
objs
1337, sw_barn02, sw_farm, 299, 0
...
end

tobj
1338, lamp_night, gen_lamps, 150, 0, 20, 6
end
```

A bare section keyword opens a block, `end` closes it, `#` begins a comment, and rows are
comma-separated. There is no header, no version and no count — the parser is driven entirely by the
section keywords it recognises.

## 2. Section census

All 59 files:

| Section | Rows | Fields per row | Meaning |
|---|---:|---|---|
| `objs` | 14,052 | **5** (one row has 6) | static objects |
| `peds` | 276 | 14 (one has 13) | pedestrians |
| `cars` | 212 | **15 / 14 / 12 / 11** | vehicles |
| `tobj` | 160 | **7** | timed objects |
| `2dfx` | 97 | **38** | effect definitions |
| `anim` | 54 | 6 | animated objects |
| `weap` | 50 | 7 | weapons |
| `txdp` | 38 | 2 | TXD parent links |
| `hier` | 35 | 5 | cutscene hierarchies |

✅ *Verified.*

**Fixed-width sections.** `objs` is 5 fields on 14,051 of 14,052 rows; `tobj` is 7 on all 160; `2dfx` is
38 on all 97. A parser can assert these.

**`cars` is genuinely variable** — 15, 14, 12 and 11 fields occur. The file's own comment explains it:

```
# cars have two extra fields wheelmodelId and wheel scale
# planes have one extra field model id of low level of detail
```

So the row length is a function of the vehicle *type* field, not a formatting inconsistency. A parser
must branch on type rather than assume a width — and the four distinct widths are exactly what a
by-type layout predicts.

**`tobj` = `objs` + 2.** Seven fields against five: a timed object is a static object plus a
time-on/time-off pair. That relationship is visible in the field counts alone.

## 3. The ID space, confirmed from the data side

Collecting the first field of every `objs`, `tobj` and `anim` row:

| Measurement | Value |
|---|---:|
| Object definitions | **14,266** |
| Lowest ID | 320 |
| Highest ID | **18,630** |
| **IDs ≥ 20000** | **0** |

✅ *Verified.* Not one definition crosses into the TXD range.

This confirms [C3.1](../C3-Model-Stores/01-the-partition.md)'s boundary from the opposite direction.
There, `cmp esi, 0x4e20` was read out of the dispatcher — a fact about code. Here, 14,266 rows authored
by level designers all fall below it — a fact about content. Neither was derived from the other.

**14,266 definitions in 20,000 slots — 71.3 % used**, leaving 5,734 free IDs below the boundary. That
headroom is why adding new objects to San Andreas is straightforward while adding a new *collision
archive* is not ([C6.3 §4](../C6-Collision/03-colstore-and-binding.md), four spare slots).

The gap from 0 to 319 is the reserved low range — vehicles and peds occupy IDs in their own bands via
the `cars` and `peds` sections.

## 4. ⚠️ Seven duplicate IDs in retail data

Seven object IDs are each defined **twice**, in two different files:

| ID | `countn2.ide` | `leveldes.ide` |
|---:|---|---|
| 16700 | `lod_rockgp1_12` | `androm_des_obj` |
| 16701 | `lod_rockgp1_05` | `china_town_gateb` |
| 16702 | `lod_rockgp2_17` | `cargo_stuff` |
| 16705 | `lod_rockgp1_09` | `cargo_test` |
| 16706 | `lod_rockgp1_07` | `carge_barrels` |
| 16707 | `lod_rockgp1_13` | `cargo_netting` |
| 16708 | `lod_rockgp2_11` | `cargo_store` |

✅ *Verified.*

The pattern is unmistakable: one file assigns a block of LOD rock models, another assigns a block of
level-design props, and the ranges overlap by seven. Note `carge_barrels` — a typo for `cargo` — in the
same block, which reads like hurried late-development work.

**Whichever file loads last wins**, and the loser's model is unreachable by ID. 🟡 *Reasoned:* since
both blocks exist in the shipped game and neither is visibly broken, either the duplicated LOD rocks
are unplaced, or the load order happens to favour the file whose objects are actually used.

⏳ **Open:** which of the two wins in practice, and whether the losing models are referenced by any IPL.
That is answerable by cross-referencing against the placement census in
[C11.2](02-ipl-placements.md) and was not done here.

**For tooling this matters:** any program that builds an ID → definition map from the IDE set must
decide what to do with a collision. Silently overwriting reproduces the game's behaviour; erroring out
rejects retail data.

## 5. Twenty-one placed-but-undefined IDs

Text IPLs place 21 model IDs that no `objs`, `tobj` or `anim` row defines.

🟡 *Reasoned:* the most likely explanation is that they are defined in a section this census did not
treat as object-defining — `hier` (35 rows, cutscene hierarchies) assigns IDs too, and cutscene objects
can legitimately be placed. A smaller possibility is genuinely dead placements referencing models cut
late.

⏳ **Open:** resolving the 21 against `hier` and the vehicle/ped ranges. Cheap, and not done here.

Either way, a loader must tolerate it: **a placement referencing an unknown ID cannot be a fatal error**,
because retail data contains 21 of them.

## 6. `txdp` — the parent link

38 rows of two fields each: a TXD name and a parent TXD name.

🟡 *Reasoned:* this is texture-dictionary inheritance — a child TXD that omits a texture falls back to
its parent's copy. That fits the name-based resolution model in
[C9.2 §2](../C9-Materials-And-Textures/02-texture-references.md), where lookup is by string and can
therefore search more than one dictionary.

At only 38 rows it is a rarely-used feature, but it means **texture resolution is not a single-dictionary
lookup** — which any tool reimplementing it needs to know.

---

### Key takeaways

- **Nine section types across 59 files**; `objs` dominates at 14,052 rows.
- Most sections are **fixed-width** and assertable; **`cars` is genuinely variable** (15/14/12/11
  fields) because row length depends on vehicle type — the file's own comment says so.
- **`tobj` is `objs` plus a time-on/time-off pair** — visible from the field counts alone.
- **14,266 definitions, IDs 320–18,630, zero at or above 20,000** — confirming
  [C3.1](../C3-Model-Stores/01-the-partition.md)'s code-derived boundary from the content side.
- **71.3 % of the model ID space is used**, leaving 5,734 free — which is why adding objects is easy and
  adding collision archives is not.
- ⚠️ **Seven IDs (16700–16708) are defined twice**, LOD rocks versus cargo props; last file loaded wins.
- **21 placed IDs have no definition** — so an unknown ID must never be a fatal error.
- `txdp` means **texture lookup can fall back to a parent dictionary**.

**Continue:** [C11.2 — IPL: the placement side](02-ipl-placements.md)
