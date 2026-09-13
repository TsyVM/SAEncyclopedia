# C11.2 — IPL: the Placement Side

> **The one-sentence version:** the same placement record in two containers — 9,315 rows of text that
> must exist before streaming starts, and 36,569 binary records that stream in with the map — plus a
> path network ten times larger than both.

[← C11.1 — IDE definitions](01-ide-definitions.md) · [Chapter 11 hub](C11-IDE-And-IPL.md) ·
[Next: C11.3 — What the placements prove →](03-what-placements-prove.md)

**Confidence:** ✅ Verified over all 53 text files and all 164 binary IPLs

---

## 1. Text IPL

Same section-keyword model as `.ide` ([C11.1 §1](01-ide-definitions.md)). The `inst` section is the
placement list:

```
inst
1337, sw_barn02, 0, 612.5, -1102.3, 78.4, 0, 0, 0.7071, 0.7071, -1
end
```

**Eleven fields, on all 9,315 rows** — no variation:

```
id, modelName, interior, x, y, z, rotX, rotY, rotZ, rotW, lod
```

✅ *Verified.* The rotation is a **quaternion**, not Euler angles — four fields, and the sampled values
are unit-length pairs like `0.7071`.

Note the row carries **both** the model ID and the model name. The name is redundant given the ID, and
that redundancy is the safety net: it lets a tool detect the ID-shift problem that
[C6.3 §2](../C6-Collision/03-colstore-and-binding.md) describes the collision loader guarding against.

### The other text sections

| Section | Rows | |
|---|---:|---|
| `path` | **165,152** | AI node network |
| `cull` | 1,266 | cull zones |
| `occl` | 1,012 | occluders |
| `enex` | 376 | enterable interiors |
| `auzo` | 155 | audio zones |
| `grge` | 52 | garages |
| `tcyc` | 8 | timecycle boxes |
| `pick` | 5 | pickups |

**`path` is 93.1 % of every text IPL row in the game** (165,152 of 177,341). The AI navigation network is an order of
magnitude larger than the object placement data it runs through — worth stating plainly, because the
visible world is what a reader assumes IPL files are *for*.

⏳ **Open:** all eight of these record layouts. `path` in particular deserves its own chapter.

## 2. Binary IPL

The 164 `.ipl` entries inside `gta3.img` ([C1.1 §3](../C1-Streaming/01-img-ver2-archive-model.md)) are
not text. They begin with a FourCC:

```c
struct BinaryIplHeader {      // 32 bytes
    char     magic[4];        // 'bnry'
    uint32_t numInst;
    uint32_t _unknown1;
    uint32_t _unknown2;
    uint32_t _unknown3;
    uint32_t numCars;
    uint32_t _unknown4;
    uint32_t instOffset;      // byte offset to the inst array
};

struct BinaryIplInst {        // 40 bytes
    float   position[3];      // +0x00
    float   rotation[4];      // +0x0C  quaternion
    int32_t modelId;          // +0x1C
    int32_t interior;         // +0x20
    int32_t lod;              // +0x24  -1 = none
};
```

✅ *Verified* across all 164 files:

| Check | Result |
|---|---|
| Files with `'bnry'` magic | **164 / 164** |
| Files where `instOffset + 40 × numInst` fits the payload | **164 / 164** |
| Placements decoded | **36,569** |

The 40-byte record and the `instOffset` field both check out on every file, with no overflow — the same
exact-fit standard used throughout this encyclopedia.

⏳ **Open:** the four unknown header dwords and the `numCars` block. `numCars` is named by its position
but its record array was not located or walked.

## 3. Text versus binary

| | Text | Binary |
|---|---:|---:|
| Files | 53 | 164 |
| `inst` placements | 9,315 | **36,569** |
| Share of world | 20 % | **80 %** |
| Distinct model IDs | 6,453 | 5,241 |
| Streamed? | no | **yes** |

🟡 *Reasoned:* the split is load-time versus stream-time. Text IPLs are parsed once at startup and hold
what must exist before streaming can run — interiors, garages, pickups, the path network. The binary
IPLs are streaming assets with their own IDs in the `25255–25510` range
([C3.1](../C3-Model-Stores/01-the-partition.md)), paged in and out with the map sections they describe.

The binary form exists because parsing 36,569 rows of comma-separated text during gameplay is not
viable; a 40-byte fixed record is a `memcpy`.

**Note the distinct-ID counts invert the placement counts.** Text places 6,453 distinct models across
9,315 placements (1.4 each); binary places 5,241 across 36,569 (7.0 each). The streamed world is built
from heavy reuse of a smaller vocabulary — the same wall, fence and road pieces repeated thousands of
times — while the text files place a wider variety of one-off objects.

## 4. The LOD link

The last field of every placement is a **LOD index**: the array index of another placement in the same
file that serves as this object's distant stand-in, or `-1`.

| Source | Has LOD | No LOD |
|---|---:|---:|
| Text | 1,419 | 7,896 |
| Binary | 4,648 | 31,921 |
| **Total** | **6,067 (13.2 %)** | **39,817** |

✅ *Verified.*

Only **13.2 %** of placements have a LOD partner. 🟡 *Reasoned:* LOD models are authored for large
distant landmarks — the `lod_rockgp1_*` names from [C11.1 §4](01-ide-definitions.md) are exactly this —
while the overwhelming majority of world objects simply stop drawing past their IDE draw distance.

Because the LOD field is an **index into the same file's placement array**, it is position-dependent:
inserting a placement anywhere in a binary IPL invalidates every LOD index after it. That is a sharp
edge for any map editor, and it is why tools append rather than insert.

## 5. Interiors

The `interior` field is not a small sequential ID:

| Value | Placements |
|---:|---:|
| 0 | 28,359 |
| 256 | 5,511 |
| 512 | 1,212 |
| 768 | 353 |
| 1024 | 266 |
| 2048 | 262 |

There are **19 distinct values** in total. The six above are the common ones and are all multiples of
256 — but **the full set is not**:

| Multiples of 256 | Non-multiples |
|---|---|
| 0, 256, 512, 768, 1024, 1280, 2048, 2304, 4096, 4352 | 1, 2, 10, 13, 14, 17, 18, 269, 2317 |
| **36,103 placements (98.7 %)** | 466 placements (1.3 %) |

⚠️ *A first draft of this page claimed "the values are multiples of 256" from the top-six table and was
wrong* — 9 of the 19 distinct values are not, and `269 = 256 + 13` and `2317 = 2304 + 13` are visibly
**sums** of a multiple of 256 and a small number.

🟡 *Reasoned:* that decomposition is the clue. The field looks like it packs **two** quantities —
something in the high bits and a small index in the low — rather than being a plain interior ID. The
1.3 % that are not clean multiples are the records where the low part is non-zero.

⏳ **Open:** confirming the packing and what each half means. The evidence points at a composite field,
but per [C10.1 §4](../C10-2dEffect/01-the-record-and-corrections.md) this chapter does not adopt a
reading it has not tested — and the lesson here is that **a top-N table is not a distribution.**

---

### Key takeaways

- Text `inst` rows are **11 fields on all 9,315** — id, name, interior, position, **quaternion**, LOD.
- The row carries **both ID and name**, and that redundancy is the defence against ID shift.
- **`path` is 93.1 % of all text IPL rows** (165,152 of 177,341) — the AI network dwarfs the visible
  world data.
- Binary IPL: `'bnry'` header plus **40-byte records**, verified on **164/164** files with no overflow,
  **36,569 placements**.
- **80 % of the world is streamed binary**, 20 % is load-time text.
- Text places 1.4 placements per distinct model; **binary places 7.0** — the streamed world is heavy
  reuse of a small vocabulary.
- Only **13.2 % of placements have a LOD partner**, and the LOD field is an **array index**, so
  inserting placements invalidates every later index.
- `interior` has **19 distinct values**; 98.7 % of placements use a multiple of 256, but 9 values are
  not — `269 = 256+13`, `2317 = 2304+13` suggest a **composite field**. Flagged, not adopted.
- ⚠️ Reading a **top-6 table as the full distribution** produced a wrong claim in this page's first
  draft — a top-N view is not a distribution.

**Continue:** [C11.3 — What the placements prove](03-what-placements-prove.md)
