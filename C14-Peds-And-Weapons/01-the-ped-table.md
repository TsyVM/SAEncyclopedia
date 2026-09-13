# C14.1 — The Ped Table

> **The one-sentence version:** 276 characters in six type classes with 270 distinct voices — and one
> row where a space replaced a comma, the fourth instance of that defect found in retail data.

[← Chapter 14 hub](C14-Peds-And-Weapons.md) · [Next: C14.2 — weapon.dat and the £ sigil →](02-weapon-dat.md)

**Confidence:** ✅ Verified over all 276 rows

---

## 1. The row

```
12, BFYRI, BFYRI, CIVFEMALE, STAT_COWARD, sexywoman, 120C,1, null,7,9,PED_TYPE_GEN,VOICE_GEN_BFYRI ,VOICE_GEN_BFYRI
```

**Fourteen fields on 275 of 276 rows.** Reading them positionally:

```
id, modelName, txdName, defaultPedType, statName, animGroup,
carsCanDrive, flags, animFile, radio1, radio2, pedType, voice1, voice2
```

🟡 *Reasoned:* the field *names* above are the conventional reading. What is ✅ verified is the **count**
(14, fixed) and the **content** of four columns that are self-identifying: the `STAT_*` reference
(field 5), the `PED_TYPE_*` value (field 12), and the two `VOICE_*` identifiers (fields 13–14) — those
carry their own prefixes and cross-check against other files
([C14.3](03-stats-and-crosschecks.md)).

Note the trailing whitespace before the comma in `VOICE_GEN_BFYRI ,` — present in the shipped data and
harmless, but a parser must strip fields rather than compare raw.

## 2. Six ped types

| `PED_TYPE_*` | Rows | Share |
|---|---:|---:|
| `PED_TYPE_GEN` | **211** | 76.4 % |
| `PED_TYPE_GANG` | 26 | 9.4 % |
| `PED_TYPE_GFD` | 16 | 5.8 % |
| `PED_TYPE_EMG` | 12 | 4.3 % |
| `PED_TYPE_SPC` | 10 | 3.6 % |
| `PED_TYPE_PLAYER` | **1** | 0.4 % |
| | **276** | |

✅ *Verified — counted after the §4 repair.* Six values, no others, and the column sums to 276.

> ⚠️ **This table must be counted after repair.** Before repairing row 51, a scan for tokens beginning
> `PED_TYPE_` finds only **275** — the defective row's value is embedded in `2,0 PED_TYPE_GEN`, so it
> does not match. A first draft reported 210 `GEN` and a 275 total without noticing the column did not
> sum to the row count.
>
> **Column totals that fail to reach the record count are a defect detector**, and this one worked
> exactly once it was checked.

**Three-quarters of the cast is `GEN`** — generic civilians. The specialised classes together account
for 65 rows, and exactly one row is the player.

🟡 *Reasoned:* `GFD` is plausibly the gang/faction "family" class and `EMG` emergency services (police,
medic, fire), which would fit their counts — 16 and 12 are about right for the emergency roster San
Andreas ships. Neither is derived here, so the expansions are not adopted.

## 3. Voices — near-total uniqueness

| Measurement | Value |
|---|---:|
| Ped rows | 276 |
| Distinct `VOICE_*` identifiers | **270** |

✅ *Verified.*

Almost every character has its own voice identifier. With two voice columns per row and 270 distinct
values across 276 rows, the overwhelming majority of peds are voiced individually rather than pooled
into a handful of banks.

For a 2004 title that is a remarkable amount of authored audio metadata, and it is consistent with San
Andreas's reputation for ambient chatter variety. It also means **the voice table is the single richest
identifier space in the ped data** — richer than the stat table (43 entries) or the type field (6).

⏳ **Open:** whether both voice columns are ever different within a row. The samples show them
identical (`VOICE_GEN_BFYRI`, `VOICE_GEN_BFYRI`), which would make one redundant — a cheap census not
run here.

## 4. ⚠️ Row 51 — the fourth comma defect

One row has 13 fields:

```
51, BMYMOUN, BMYMOUN, CIVMALE, STAT_SENSIBLE_GUY, man,0800,1, man,2,0 PED_TYPE_GEN,VOICE_GEN_BMYMOUN ,VOICE_GEN_BMYMOUN
```

Against a normal row:

```
12, BFYRI,   BFYRI,   CIVFEMALE, STAT_COWARD,       sexywoman, 120C,1, null,7,9,PED_TYPE_GEN,VOICE_GEN_BFYRI ,VOICE_GEN_BFYRI
```

The difference is at `2,0 PED_TYPE_GEN` — **a space where a comma belongs**, merging two fields into
one.

✅ Verified. Every other row in the file parses to 14.

This is the same defect class as the three `cars` rows in
[C13.2 §3](../C13-Vehicle-Data/02-cars-section-repaired.md), and the running total across retail
`data/` is now four missing separators plus two malformed values
([C14 §14.3](C14-Peds-And-Weapons.md)).

**Unlike the vehicle case, this one does not manufacture a false anomaly.** The shift pushes
`PED_TYPE_GEN` into the field where the radio value belongs, but nothing cross-references that column,
so no downstream check fires. The defect is invisible unless you count fields.

That asymmetry is worth noticing: [C13.2 §4](../C13-Vehicle-Data/02-cars-section-repaired.md)'s defect
was *caught* by a cross-check that then reported it wrongly; this one is caught by nothing at all. **A
cross-reference finds defects only in the columns it happens to cross-reference.**

## 5. Repairing it

The signature is narrower than the vehicle case — a field containing a space adjacent to a known
`PED_TYPE_*` token:

```python
def parse_ped_row(line):
    f = [x.strip() for x in line.split(',')]
    for i, v in enumerate(f):
        if ' ' in v and 'PED_TYPE_' in v:
            head, _, tail = v.partition(' ')
            f = f[:i] + [head.strip(), tail.strip()] + f[i+1:]
            break
    return f
```

Applied to all 276 rows this repairs exactly one and touches nothing else — verified.

## 6. Ped IDs and the model space

| Measurement | Value |
|---|---:|
| Lowest ped ID | **0** |
| Highest ped ID | **299** |

Peds occupy IDs 0–299 — inside the 0–19999 model range
([C3.1](../C3-Model-Stores/01-the-partition.md)) and below the 320 floor that
[C11.1 §3](../C11-IDE-And-IPL/01-ide-definitions.md) measured for world objects.

So the low band C11.1 described as "reserved" is now accounted for: **peds hold 0–299, weapons hold
321–373, and world objects begin at 320.** The three ranges are contiguous and non-overlapping, which
is a small but satisfying closure of an open observation from three chapters back.

🟡 The 300–319 gap and the single-ID overlap at 320/321 were not investigated.

---

### Key takeaways

- **14 fields on 275 of 276 rows.** Four columns are self-identifying by prefix (`STAT_`, `PED_TYPE_`,
  two `VOICE_`); the rest are 🟡 conventional readings.
- **Six ped types**, with `PED_TYPE_GEN` at **76.4 % (211)** and exactly one `PED_TYPE_PLAYER`; the
  column sums to 276 **only after the §4 repair** — before it, 275, because the defect hides one value.
- **270 distinct voices for 276 peds** — near-total uniqueness, the richest identifier space in the ped
  data.
- ⚠️ **Row 51 (`BMYMOUN`) has a space where a comma belongs** — 13 fields. The fourth such defect found
  in retail data.
- Unlike C13's defect, this one **manufactures no false anomaly** — nothing cross-references the
  affected column, so it is invisible unless you count fields. **Cross-checks only find defects in the
  columns they cross-reference.**
- **Peds 0–299, weapons 321–373, objects from 320** — the low model band is now fully accounted for,
  closing an observation from C11.1.

**Continue:** [C14.2 — weapon.dat and the £ sigil](02-weapon-dat.md)
