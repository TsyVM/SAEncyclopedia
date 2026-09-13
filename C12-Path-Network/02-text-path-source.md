# C12.2 — The Text Path Source

> **The one-sentence version:** 12,704 groups of exactly twelve rows — a fixed-slot format where a
> third of the slots are empty, and the emptiness is the format working as designed.

[← C12.1 — NODES\*.DAT](01-nodes-dat.md) · [Chapter 12 hub](C12-Path-Network.md) ·
[Next: C12.3 — Two representations →](03-two-representations.md)

**Confidence:** ✅ Verified over all 5 files

---

## 1. The shape

`path` sections appear in exactly **five** files — `paths.ipl` through `paths5.ipl` — and hold two row
widths and nothing else:

| Row width | Count |
|---:|---:|
| 2 fields | 12,704 |
| 13 fields | 152,448 |

```
12,704 × 12 = 152,448
```

✅ **Exact.** Every 2-field row is a group header followed by precisely twelve 13-field rows. Not
"usually twelve" — twelve, 12,704 times.

That single multiplication is the whole structural proof. A format with variable group sizes could not
produce an exact 12:1 ratio across 165,152 rows.

```
path
2, -1                                            <- group header: type 2, id -1
2, 2, 0, -33017.1, 21011.3, 97.6, 0, 1, 1, 1, 0, 1, 0
2, -1, 0, -36936.3, 21672.3, 96.6, 0, 1, 1, 1, 0, 1, 0
...  (twelve rows total)
end
```

## 2. Group types

The header's first field takes three values:

| Type | Groups |
|---:|---:|
| 0 | 6,331 |
| 1 | 5,929 |
| 2 | 444 |

✅ *Verified.*

🟡 *Reasoned:* types 0 and 1 are near-equal in count and dominate; type 2 is a small special case. Given
the binary file's split into **30,587 vehicle nodes and 37,650 pedestrian nodes**
([C12.1 §1](01-nodes-dat.md)), the natural reading is that 0 and 1 are the vehicle and pedestrian
networks. The counts do not map one-to-one — 6,331 and 5,929 groups against 30,587 and 37,650 nodes —
because a group holds up to twelve nodes and many slots are empty (§3).

⏳ **Open:** which type is which, and what type 2 is. Determining it means correlating a group's
coordinates against the vehicle/ped split in the binary files, which was not done here.

## 3. A third of the slots are empty

Of the 152,448 body rows:

| | Count | Share |
|---|---:|---:|
| Live (non-zero position) | **100,089** | 65.7 % |
| All-zero placeholders | **52,359** | 34.3 % |

✅ *Verified.*

**Every group reserves twelve slots and most do not fill them.** 34.3 % of the file is padding.

🟡 *Reasoned:* this is a fixed-capacity authoring format — a path group is a struct with twelve node
slots, written out whole regardless of how many are used. The same design as the pools in
[C4](../C4-Entities-And-Pools/C4-Entities-And-Pools.md): fixed capacity, occupancy tracked separately.

**Consequence for parsing:** a reader must skip all-zero rows rather than treat them as nodes at the
origin. A tool that takes them literally places 52,359 phantom nodes at (0, 0, 0) — which is inside the
world, so nothing would look obviously wrong.

## 4. Coordinates are not world units

The raw values are large:

```
X  −47873.1 … 47140.8
Y  −46922.5 … 45661.4
Z    −737.7 … 32373.9
```

Nearly **16×** the world's extent. Dividing by 16:

```
X  −2992.07 … 2946.30
Y  −2932.66 … 2853.84
```

which is the world, and — as [C12.3](03-two-representations.md) shows — the binary node extents to
within 0.2 units.

**The text stores positions in ⅟₁₆-unit fixed point.** A parser that reads the field as world units
places everything sixteen times too far out, and since the values are decimal-formatted floats there is
nothing in the file to signal it.

## 5. Live nodes versus binary nodes

| Source | Nodes |
|---|---:|
| Text, live | **100,089** |
| Binary (`NODES*.DAT`) | **68,237** |

The text carries **31,852 more** live nodes than the binary.

🟡 *Reasoned:* the binary files are a compiled product, not a copy. Some text nodes are filtered,
merged, or belong to networks the runtime does not need in that form. The direction is what you would
expect — a source set larger than its compiled output — and the reverse would be alarming.

⏳ **Open:** what the 31,852 difference consists of. Answering it means matching text nodes to binary
nodes by position, which is tractable given both are now decoded and was not attempted here.

## 6. Why the text still ships

If the binary is what the runtime reads ([C12.1](01-nodes-dat.md)), the text is redundant at runtime —
yet 165,152 rows of it are in the shipped game, in files the streaming system parses at startup.

🟡 *Reasoned:* it ships because the IPL loader parses whole files and these sections sit inside files
that also carry `inst`, `cull` and `occl` data the game does need. Splitting them out would have meant
touching the map export pipeline late in development.

That reading is consistent with the retail data's other rough edges — the seven duplicate IDs and the
`carge_barrels` typo in [C11.1 §4](../C11-IDE-And-IPL/01-ide-definitions.md).

---

### Key takeaways

- **12,704 group headers × 12 = 152,448 body rows, exactly.** A fixed-slot format, proved by one
  multiplication over 165,152 rows.
- Three group types: **0 (6,331), 1 (5,929), 2 (444)**; 🟡 likely vehicle / pedestrian / special, ⏳ not
  confirmed.
- **34.3 % of slots are all-zero padding.** A parser that takes them literally creates 52,359 phantom
  nodes at the origin — inside the world, so it would not look wrong.
- Coordinates are **⅟₁₆-unit fixed point**, ~16× the world extent, with nothing in the file to signal it.
- The text holds **100,089 live nodes against the binary's 68,237** — a source set larger than its
  compiled output, which is the expected direction.
- 🟡 It still ships because these sections live inside IPL files the game needs for other reasons.

**Continue:** [C12.3 — Two representations of one network](03-two-representations.md)
