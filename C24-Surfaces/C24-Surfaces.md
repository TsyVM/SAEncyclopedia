# Chapter 24 — Surface Materials: Friction, Physics and Audio

> **Goal of this chapter:** decode how San Andreas describes what the world is *made of* — the 179 surface
> types and the physics, audio and procedural behaviour keyed to each — and pin every claim down with
> arithmetic and cross-file agreement. The headline is the project's strongest evidence in its purest form:
> the surface-type namespace of **179** names is agreed by **three independent sources** — `surfinfo.dat`,
> `surfaud.dat` and the executable's own hard-coded string table — with the two data files in **identical
> order**.

**Subsystem category:** Physics / world
**Depends on:** [C6 — Collision](../C6-Collision/C6-Collision.md) (collision points carry a surface type) ·
[C13 — Handling & Vehicle Data](../C13-Vehicle-Data/C13-Vehicle-Data.md) (tyre grip overrides friction)
**Ties:** [C6](../C6-Collision/C6-Collision.md), [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md), [C20](../C20-Audio/C20-Audio.md), [C21](../C21-Particles/C21-Particles.md), [C22](../C22-Map-Zones/C22-Map-Zones.md)
**RE status:** Verified
**Confidence:** ✅ Verified (the 179-type namespace and its three-way agreement, both record grammars, the
friction matrix, the procedural subsets) / ⏳ (the numeric semantics of a few physics scalars)

---

## Deep-dive pages

- [C24.1 — The 179-surface namespace](01-the-surface-namespace.md): one list of surface-type names, agreed
  to the entry by `surfinfo.dat`, `surfaud.dat` and the executable — and the procedural files that key off
  a subset of it.
- [C24.2 — `surfinfo.dat`: the 37-field physics record](02-surfinfo-the-physics-record.md): a fixed
  37-column record — adhesion group, tyre grip, and 30 behaviour flags — uniform across all 179 rows, with
  every enumerated field staying inside the domain the file's own header documents.
- [C24.3 — `surface.dat` and `surfaud.dat`](03-friction-matrix-and-audio.md): a 6 × 6 lower-triangular
  adhesion/friction matrix whose row labels are exactly the six adhesion groups, and a 9-column boolean
  audio-category table.
- [C24.4 — Surfaces at runtime and modding](04-surfaces-modding-and-runtime-use.md): the surface ID
  path from COL face through `CColPoint` to `CPhysical` friction + audio selection; gang territory flag;
  C47 traction path; `surfinfo.dat` as a single-file global physics modifier; ID namespace limit (0–178).

---

## 24.1 The result first

| Claim | Evidence |
|---|---|
| There are **179 surface types** | `surfinfo.dat` and `surfaud.dat` each have exactly 179 records ✅ |
| **Three-way agreement** on the namespace | all 179 names are hard-coded strings in `gta_sa.exe` — `179 / 179` ✅ |
| The two data files are in **identical order** | `surfinfo` and `surfaud` list the same names in the same sequence ✅ |
| `surfinfo` is a **fixed 37-column record** | `179 / 179` records have exactly 37 whitespace columns ✅ |
| The surface name is **repeated as the last column** | `179 / 179` (the same self-labelling seen in C22 zones) ✅ |
| `surfaud` is a **10-column** record (9 boolean flags) | `179 / 179` at 10 columns; every flag `∈ {0, 1}` ✅ |
| `surface.dat` is a **6 × 6 lower-triangular** matrix | rows of `1, 2, 3, 4, 5, 6` = **21** values ✅ |
| Its six rows == the six **adhesion groups** | `{RUBBER, HARD, ROAD, LOOSE, SAND, WET}` in both files ✅ |
| Enumerated fields obey the header's **documented domains** | skidmark, friction-effect and bullet-fx all in range ✅ |
| Procedural files reference **only defined surfaces** | `procobj` (17) and `plants` (42) refs ⊆ the 179 ✅ |

## 24.2 One namespace, five files

San Andreas splits "what a surface is like" across five files, each owning one concern, and binds them
together by a shared vocabulary of surface-type names rather than by numeric IDs in the data:

```
                 file            what it owns                         keyed by
surfinfo.dat   179 records   physics: friction, flags, effects      surface name
surfaud.dat    179 records   audio: 9 material categories           surface name (same order)
surface.dat    6×6 matrix    adhesion-group friction coefficients   adhesion group
procobj.dat    rules         procedural objects on a surface        surface name (subset)
plants.dat     rules         procedural plants on a surface         surface name (subset)
```

The binding is checkable, and it checks out. `surfinfo` and `surfaud` do not merely have the same *number*
of rows — they carry the identical list of names in the identical order, so a loader can pair the physics
and audio of a surface by row index without a lookup. `surfinfo`'s adhesion-group column draws from exactly
the six labels that head `surface.dat`'s matrix. And every surface named by the two procedural files is one
of the 179 defined in `surfinfo`. Five files, one namespace, no dangling reference.

## 24.3 Why the three-way agreement matters

[C22.1](../C22-Map-Zones/01-the-zone-tables.md) established the project's strongest form of evidence:
information that reproduces independently across subsystems cannot be at the wrong offset or the wrong
count. The surface namespace is that argument in its cleanest form yet. The count **179** is stated three
times, by three parties who would have no reason to agree unless they were describing the same thing: the
physics table, the audio table, and the compiled executable, which carries all 179 names as literal strings
so it can map a surface name to its internal index at load time. No community list is consulted or needed —
the game's own files triangulate the answer.

## 24.4 What this chapter does not claim

⏳ The **numeric meaning of some physics scalars** is not fully pinned. `TYRE_GRIP` and `WET_GRIP` are
labelled by the header and their ranges are consistent with a grip multiplier and a wet-weather modifier,
but this chapter does not derive their exact application in the handling model — that is
[C13](../C13-Vehicle-Data/C13-Vehicle-Data.md)'s territory. The **30 behaviour flags** are read positionally
and their domains verified; a handful (e.g. `ROUGHNESS`, `FLAME`) are small integer scales whose upper
bounds are documented in the header but not independently confirmed against the engine.

⏳ The **`surface.dat` coefficients themselves** are taken as authored values. That the matrix is a
6 × 6 lower triangle of adhesion pairs is ✅; whether a given cell is, say, a Coulomb friction coefficient
or a scaled analogue is not derived from the data.

---

### Key takeaways

- ✅ **179 surface types**, agreed by **three independent sources** — `surfinfo.dat`, `surfaud.dat` and the
  executable's hard-coded string table — with the two data files in **identical order**.
- ✅ `surfinfo.dat` is a uniform **37-column** physics record (adhesion group, tyre/wet grip, skidmark,
  friction effect, 30 behaviour flags), with the surface name **repeated as the 37th column**.
- ✅ `surfaud.dat` is a **10-column** record — the name plus **9 boolean** audio-material categories.
- ✅ `surface.dat` is a **6 × 6 lower-triangular** adhesion/friction matrix (21 values) whose six rows are
  exactly the adhesion groups `surfinfo` references.
- ✅ Every enumerated field stays inside the domain the file's **own header** documents; procedural
  `procobj`/`plants` reference **only** defined surfaces.
- ⏳ The exact numeric semantics of a few physics scalars and the friction coefficients are left to
  [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md) and future work.

**Continue:** [C24.1 — The 179-surface namespace](01-the-surface-namespace.md)

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C6](../C6-Collision/C6-Collision.md), [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md), [C20](../C20-Audio/C20-Audio.md), [C21](../C21-Particles/C21-Particles.md), [C22](../C22-Map-Zones/C22-Map-Zones.md)
- **Known bugs / gotchas:** surface count must agree 3 ways (surfinfo=surfaud=exe); procobj/plants must be subset.
- **Modding:** surface.dat grip matrix + surfinfo flags tune vehicle/ped surface behaviour (C47).
- **Performance:** surface lookup per wheel/foot contact.
