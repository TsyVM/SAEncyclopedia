# C14.2 — weapon.dat and the £ Sigil

> **The one-sentence version:** three record types selected by a leading symbol, one of which is a
> pound sign — the only byte above 127 in the entire file, and a hard dependency on latin-1 decoding
> that nothing in the file declares.

[← C14.1 — The ped table](01-the-ped-table.md) · [Chapter 14 hub](C14-Peds-And-Weapons.md) ·
[Next: C14.3 — Stats and cross-checks →](03-stats-and-crosschecks.md)

**Confidence:** ✅ Verified

---

## 1. Sigil dispatch, again

`weapon.dat` uses the same mechanism as `handling.cfg` ([C13.1 §1](../C13-Vehicle-Data/01-handling-cfg.md)) —
whitespace-separated fields, record type chosen by the first token — but with a different sigil set:

| Sigil | Records | Fields | Example subject |
|---|---:|---|---|
| `$` | **53** | 25 (43 rows) / 29 (10 rows) — see [C14.4](04-the-gun-record-columns.md) | `GRENADE`, projectiles and firearms |
| `%` | **21** | 10 | `python` — per-weapon secondary block |
| `£` | **17** | 12 | `UNARMED`, `MELEE` |
| `ENDWEAPONDATA` | 1 | 1 | terminator |

✅ *Verified* over all 91 data rows.

```
£ UNARMED     MELEE       10.0  1.6   -1  -1   0   UNARMED   4    1   null
$ GRENADE     PROJECTILE  30.0 40.0  342  -1   8   grenade   1   75  0.0 0.0 0.0  1 0 1.0 1.0 0 99 6 0 99 8 9
% python      0.1   0.5   0.0   0.0  254  633  254  633
```

Unlike `handling.cfg`, whose every record type is fixed-width, **`$` records come in two widths** — 43
at 25 fields and 10 at 29 ([C14.4](04-the-gun-record-columns.md) re-counts this precisely and derives the
column layout). The wider variant carries four extra floats — `speed, radius, lifespan, spread` — on
exactly the thrown/area-effect weapons.

The file ends with an explicit `ENDWEAPONDATA` terminator — the only data file in this encyclopedia that
declares its own end rather than relying on EOF.

## 2. The `£`

**Byte `0xA3` is the only value above 127 in the whole file.**

```
weapon.dat size: 11,585 bytes
non-ASCII bytes present: [163]        # 0xA3 = £ in latin-1 / cp1252
```

✅ Verified by scanning every byte.

It appears **19** times: **17 as the leading sigil of a live melee record**, and twice inside comments —

```
# MELEE DATA (use £ identifier)
#£ SKATEBOARD	MELEE	10.0  1.6	340	-1		1		SKATEBOARD	1			1		null
```

The second is a **commented-out weapon**: a `SKATEBOARD` melee record, fully written and disabled with a
leading `#`. 🟡 *Reasoned:* cut content left in place — the same residue as the nine orphaned ped stats
in [C14.3 §2](03-stats-and-crosschecks.md) and the six unused handling entries in
[C13 §13.3](../C13-Vehicle-Data/C13-Vehicle-Data.md).

⚠️ A first draft of this page said "exactly 17 times" — counting sigils rather than bytes. The
distinction matters here precisely because the two extra occurrences are the interesting ones.

### Why this matters more than it looks

The file carries **no encoding declaration**. A parser that opens it as UTF-8 hits `0xA3` as an invalid
continuation byte and either throws or substitutes a replacement character — and in the latter case the
sigil no longer matches `£`, so **all 17 melee weapons silently vanish** from the parsed set.

That is the same silent-loss failure mode as the two-root DFFs in
[C7.1 §3](../C7-RenderWare-Stream/01-the-section-stream.md) and the `peds.col` walk in
[C6.2 §4](../C6-Collision/02-header-and-bounds.md): no error, no crash, a third of the weapon table just
missing.

**The fix is one line** — decode as `latin-1` (or `cp1252`), never UTF-8, for every file in `data/`.
`latin-1` is the safe default because it maps every byte 0–255 to a codepoint and therefore cannot fail
on any input.

🟡 *Reasoned:* the choice of `£` is almost certainly incidental — a Rockstar North developer picking a
key on a UK keyboard for a marker that needed to be visually distinct from `$` and `%`. It is a small
artefact of where the game was made, frozen into the data format.

## 3. Three sigils, three roles

🟡 *Reasoned* from the samples and counts:

- **`£` — melee.** `UNARMED` and `MELEE` appear as the second token; 17 records for fists, bats,
  knives, chainsaw and the rest. 12 fields.
- **`$` — ranged.** `GRENADE`/`PROJECTILE`, with 25–29 fields covering damage, range, model IDs,
  ammo and animation. 53 records — the bulk of the arsenal.
- **`%` — per-weapon secondary data.** 21 records, 10 fields, keyed by a **lowercase** name (`python`)
  rather than the uppercase identifiers the other two use.

That casing difference is the most useful structural signal in the file: `£` and `$` records key on
`UPPERCASE` weapon identifiers, while `%` records key on `lowercase` model names. They are indexing two
different namespaces, which is the same ID-versus-name split C13 found across the vehicle files
([C13.1 §4](../C13-Vehicle-Data/01-handling-cfg.md)).

⏳ **Open:** confirming the `%` namespace against the `weap` IDE section's model names. 21 `%` records
against 50 `weap` rows means it covers a subset, and identifying which subset would settle what the
section is for. Cheap; not done here.

## 4. The `weap` IDE section

50 rows, **7 fields on every one** — the cleanest section in the entire data set, with no defects and no
width variation:

```
321, gun_dildo1, gun_dildo1, null, 1, 50, 0
```

```
id, modelName, txdName, animFile, ?, ?, ?
```

IDs run **321–373**. Combined with peds at 0–299 and objects from 320
([C14.1 §6](01-the-ped-table.md)), the low model band is fully accounted for.

Note the model names are ordinary object names (`gun_dildo1`, `flowera`, `gun_cane`) — weapons are
models like any other, and it is the `weap` section that marks them as weapons rather than anything in
the model itself.

## 5. `melee.dat`

153 rows, all keyed by an alphabetic first token, in two dominant widths:

| Width | Rows |
|---:|---:|
| 2 | 52 |
| 8 | 52 |
| 1 | 16 |
| 9 | 13 |

The 1-field rows are block keywords such as `START_LEVELS`. The 52/52 split between 2- and 8-field rows
suggests **paired records** — a header and a data line per entry, 52 of each.

⏳ **Open.** The block structure was not decoded. The 52/52 symmetry is a strong hint and is recorded as
the starting point.

---

### Key takeaways

- `weapon.dat` uses **sigil dispatch** like `handling.cfg`, but with `$` (53), `%` (21) and `£` (17),
  plus an explicit `ENDWEAPONDATA` terminator — the only self-terminating data file here.
- **`£` = byte `0xA3` is the only non-ASCII byte in the file** — **19 occurrences: 17 live melee sigils
  plus 2 in comments**, one of which is a fully-written, commented-out `SKATEBOARD` weapon.
- **Decoding as UTF-8 silently drops all 17 melee weapons** — no error, no crash. Use `latin-1` for every
  file in `data/`; it cannot fail on any byte.
- `$` records come in **two widths** (25 and 29 — see [C14.4](04-the-gun-record-columns.md)) — unlike
  `handling.cfg`, where every type is fixed-width.
- **`£`/`$` key on UPPERCASE identifiers, `%` keys on lowercase model names** — two namespaces in one
  file, the same ID-versus-name split C13 found in the vehicle data.
- The **`weap` IDE section is the cleanest in the data set** — 50 rows, 7 fields, zero defects. IDs
  321–373.
- ⏳ `melee.dat`'s 52/52 two-and-eight-field symmetry suggests paired records; not decoded.

**Continue:** [C14.3 — Stats and cross-checks](03-stats-and-crosschecks.md)
