# C15.1 — Twenty-Three Weathers, Eight Hours

> **The one-sentence version:** a perfectly regular grid — 23 blocks, 8 rows each, identical labels
> throughout — whose only irregularity is which regions got how many weathers.

[← Chapter 15 hub](C15-Timecycle.md) · [Next: C15.2 — The columns the file names itself →](02-self-documenting-columns.md)

**Confidence:** ✅ Verified over the full file

---

## 1. The block structure

```
//////////// EXTRASUNNY_LA
//Amb   Amb_Obj   Dir   Sky top   Sky bot   ...        <- column header
//Midnight
22 22 22   220 212 130   255 255 255   0 23 24   ...
//5AM
22 22 22   194 194 142   255 255 255   0 20 20   ...
//6AM
...
//10PM
...
//
//////////// SUNNY_LA
```

A block opens with a `////////////` banner naming the weather, and each data row is preceded by a
comment naming its time of day.

| Measurement | Value |
|---|---:|
| Total lines | 437 |
| Comment lines (`//`) | 253 |
| Data rows | **184** |
| Weather blocks | **23** |
| Rows per block | **8 — in all 23** |

✅ *Verified.* `23 × 8 = 184`, exactly.

**Every block carries the identical time-label sequence**, with no exceptions:

```
Midnight · 5AM · 6AM · 7AM · Midday · 7PM · 8PM · 10PM
```

Only one distinct sequence exists across all 23 blocks — checked as a tuple comparison, not by eye.

## 2. The sampling is deliberately uneven

Eight samples across 24 hours, spaced like this:

```
00:00   05:00 06:00 07:00        12:00              19:00 20:00     22:00
  |       |     |     |            |                  |     |         |
  └─ 5h ──┴─1h──┴─1h──┴──── 5h ────┴─────── 7h ───────┴─1h──┴── 2h ───┘
```

**Three samples in the two hours around dawn, three in the three hours around dusk, and single points
at midday and midnight.** The gaps are 5h, 1h, 1h, 5h, 7h, 1h, 2h.

🟡 *Reasoned:* the engine interpolates between adjacent rows, so sample density is a direct statement
about where the lighting changes fastest. Dawn and dusk get 1-hour resolution because sky colour swings
hard there; the 7-hour gap from midday to 7PM is a period where nothing much happens.

That is an authoring decision visible entirely in the row labels, without reading a single value.

## 3. Weather is per-region

| Region | Blocks | Names |
|---|---:|---|
| **Los Santos** | 5 | `EXTRASUNNY_LA`, `SUNNY_LA`, `EXTRASUNNY_SMOG_LA`, `SUNNY_SMOG_LA`, `CLOUDY_LA` |
| **San Fierro** | 5 | `SUNNY_SF`, `EXTRASUNNY_SF`, `CLOUDY_SF`, `RAINY_SF`, `FOGGY_SF` |
| **Countryside** | 4 | `EXTRASUNNY_`, `SUNNY_`, `CLOUDY_`, `RAINY_COUNTRYSIDE` |
| **Las Venturas** | 3 | `SUNNY_`, `EXTRASUNNY_`, `CLOUDY_VEGAS` |
| **Desert** | 3 | `EXTRASUNNY_`, `SUNNY_`, `SANDSTORM_DESERT` |
| **Special** | 3 | `UNDERWATER`, `EXTRACOLOURS_1`, `EXTRACOLOURS_2` |

✅ *Verified* by suffix census.

The distribution is the design:

- **Only Los Santos has smog.** Two of its five weathers are `SMOG` variants — a per-region atmospheric
  effect nowhere else in the game.
- **Only San Fierro has fog**, and it is one of only two regions with rain.
- **Only the desert has a sandstorm.**
- **Las Venturas has the fewest** at three, and no precipitation at all — appropriate for a desert
  resort city.

This is not a global weather system with regional tinting. Each region has an **independently authored
set**, and even the *count* varies. A weather is a `(region, condition)` pair, not a condition applied
to a region.

## 4. The three special blocks

`UNDERWATER`, `EXTRACOLOURS_1` and `EXTRACOLOURS_2` are not weathers in the ordinary sense — they carry
no region suffix and describe conditions rather than skies.

🟡 *Reasoned:* `UNDERWATER` is the submerged colour grade, which explains why it needs the same 51
columns as a weather — it is applied through the same pipeline. The two `EXTRACOLOURS` blocks are
plausibly script-selected overrides for set-piece moments.

⏳ **Open:** what selects them. That is a code question, not a data one.

Note they still carry the full 8-row time structure, so even underwater has a dawn and a dusk.

## 5. Counting it

```python
import re, collections

def timecyc_blocks(path):
    blocks, rows, labels = [], collections.defaultdict(list), collections.defaultdict(list)
    current, last_label = None, None
    for line in open(path, encoding='latin-1'):        # latin-1, per C14.2 §2
        s = line.strip()
        if s.startswith('////////////'):
            current = s.strip('/').strip()
            blocks.append(current)
        elif s.startswith('//'):
            last_label = s[2:].strip()
        elif s:
            rows[current].append(re.split(r'\s+', s))
            labels[current].append(last_label)
    return blocks, rows, labels
```

Two assertions worth keeping, both of which hold on retail data:

```python
assert all(len(rows[b]) == 8 for b in blocks)          # 23/23
assert all(len(r) == 51 for b in blocks for r in rows[b])   # 183/184 — see C15.3
```

The second one **fails** on the shipped file, by design of this chapter: it is exactly what catches the
defect in [C15.3](03-the-fifth-defect.md). A parser should treat that failure as a warning about one
known row rather than a reason to reject the file.

---

### Key takeaways

- **437 lines: 253 comments, 184 data rows** = **23 blocks × 8 rows**, exact.
- **All 23 blocks share one identical time-label sequence** — verified by tuple comparison, not
  inspection.
- Sampling is **deliberately uneven** — 1-hour resolution at dawn and dusk, a 7-hour gap across the
  afternoon. Authoring intent readable from labels alone.
- **Weather is per-region and independently authored**: only LA has smog, only SF has fog, only the
  desert has a sandstorm, and block counts differ by region.
- Three special blocks — `UNDERWATER` and two `EXTRACOLOURS` — use the same 51-column format and still
  carry all 8 time rows.
- The `len(r) == 51` assertion **fails on retail data** by exactly one row, which is the point of
  [C15.3](03-the-fifth-defect.md).

**Continue:** [C15.2 — The columns the file names itself](02-self-documenting-columns.md)
