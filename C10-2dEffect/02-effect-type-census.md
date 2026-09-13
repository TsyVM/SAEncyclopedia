# C10.2 — The Effect-Type Census

> **The one-sentence version:** eight effect types, and every one of them has exactly one payload size
> across all 17,395 records in the game — a uniformity strong enough to build a parser on without
> reading a line of code.

[← C10.1 — The record, and two corrections](01-the-record-and-corrections.md) ·
[Chapter 10 hub](C10-2dEffect.md) · [Next: C10.3 — The light record →](03-the-light-record.md)

**Confidence:** ✅ Verified (types, sizes) / 🟡 (names)

---

## 1. The table

Full population — every `2dEffect` entry in `gta3.img`:

| Type | Entries | Share | `dataSize` | Reading |
|---:|---:|---:|---:|---|
| **9** | **14,908** | **85.7 %** | 12 | cover point |
| 0 | 1,038 | 6.0 % | 80 | **light / corona** |
| 3 | 793 | 4.6 % | 56 | ped attractor |
| 7 | 489 | 2.8 % | 88 | roadsign |
| 6 | 78 | 0.4 % | 44 | enter / exit marker |
| 1 | 54 | 0.3 % | 24 | particle emitter |
| 8 | 30 | 0.2 % | 4 | trigger point |
| 10 | 5 | 0.0 % | 40 | escalator |
| | **17,395** | | | |

✅ **Every type has exactly one `dataSize`.** Not a common value with outliers — one value, on every
instance, game-wide.

That is the finding that makes this table usable. A type field whose payload size varies would tell you
nothing; a type field that selects a fixed layout every single time is a discriminated union, and a
parser can dispatch on it safely.

🟡 The **names** are the community reading. What is ✅ is the type number, the size, and the count. A
parser should switch on the number and treat the name as a comment — the same discipline as
[C3.1 §1](../C3-Model-Stores/01-the-partition.md), where ranges are verified and labels are convention.

## 2. Type 9 is the world

**85.7 % of every effect record in San Andreas is type 9**, at 12 bytes each — three dwords.

Reading it as cover points, the number says something about the game that no code inspection would:
the map is annotated with roughly **fifteen thousand hand-or-tool-placed spatial hints** for the AI,
against a thousand lights and eight hundred ped attractors. The environment art carries far more
information for the AI than for the renderer.

At 12 bytes with a 20-byte entry header, each cover point costs 32 bytes on disk — 477 KB across the
game, which is why they could be afforded in that quantity.

⏳ **Open:** the three dwords. This is the highest-value remaining target in the chapter: 14,908
instances, only 12 bytes to account for, and the density means any hypothesis is testable immediately.

## 3. The long tail is where the character is

The rare types are the interesting ones for anyone modifying the world:

- **Type 10 — escalators, 5 in the entire game.** Five records, 40 bytes each. A whole effect type
  implemented for a handful of placements.
- **Type 8 — 30 records at 4 bytes.** The smallest payload; a single dword.
- **Type 1 — particle emitters, 54.** Fewer than expected, which suggests most particle effects are
  spawned by script or code rather than placed on geometry.

🟡 *Reasoned:* the low particle count relative to lights (54 vs 1,038) is consistent with 2dEffect being
the *static placement* channel — things bolted to a model — while dynamic effects come from elsewhere.

## 4. Distribution across models

1,681 of 12,955 DFFs carry any effect at all — **13.0 %**. Those 1,681 models hold 17,395 entries,
averaging **10.3 effects per model that has any**.

So effects are not sprinkled thinly; they cluster heavily. A model either has none (87 %) or has ten.
🟡 *Reasoned:* that is what you would expect if effects are authored per-building rather than
per-surface — a nightclub gets its whole lighting rig in one model, a plain wall gets nothing.

## 5. Parsing it

```python
EFFECT_SIZES = {0: 80, 1: 24, 3: 56, 6: 44, 7: 88, 8: 4, 9: 12, 10: 40}

def effects(buf, off, end):
    count = struct.unpack_from('<I', buf, off)[0]
    off += 4
    for _ in range(count):
        x, y, z, etype, dsize = struct.unpack_from('<3fII', buf, off)
        expected = EFFECT_SIZES.get(etype)
        if expected is not None and dsize != expected:
            raise ValueError(f'type {etype} expected {expected} bytes, got {dsize}')
        yield (x, y, z), etype, buf[off + 20 : off + 20 + dsize]
        off += 20 + dsize
    # off must now equal `end` — true for 1,681/1,681 retail sections
```

The `dsize != expected` check is worth keeping. It costs nothing, it held on every retail record, and it
turns a corrupt or unknown-type section into an error rather than a desynchronised walk — the failure
mode that has bitten this project three times already
([C6.2 §4](../C6-Collision/02-header-and-bounds.md),
[C7.1 §3](../C7-RenderWare-Stream/01-the-section-stream.md),
[C10.1 §3](01-the-record-and-corrections.md)).

## 6. Types absent from the data

Types **2**, **4** and **5** do not appear in `gta3.img` at all.

🟡 *Reasoned:* the type space is not contiguous in use, which mirrors `COL4`
([C6.1 §3](../C6-Collision/01-col-container-and-versions.md)) — engine support that the shipped content
never exercised. Whether the loader handles them is a code question this chapter does not answer.

⏳ **Open:** whether types 2/4/5 appear in `gta_int.img`, `player.img` or `cutscene.img`. This census
covered `gta3.img` only; extending it is trivial and was not done.

---

### Key takeaways

- **Eight types, each with exactly one `dataSize`** across all 17,395 records — a true discriminated
  union, safe to dispatch on.
- Type numbers, sizes and counts are ✅; the **names are 🟡 convention** and a parser should switch on
  the number.
- **Type 9 is 85.7 % of all effects** — the map carries far more AI annotation than lighting.
- Effects **cluster**: 13 % of models carry any, and those average 10.3 each.
- Types **2, 4 and 5 never appear** in `gta3.img` — engine capability the content did not use.
- Validate `dataSize` against the type table while parsing; it is free and it converts desynchronisation
  into an error.
- ⏳ Type 9's three dwords are the highest-value remaining target: 14,908 samples, 12 bytes.

**Continue:** [C10.3 — The light record](03-the-light-record.md)
