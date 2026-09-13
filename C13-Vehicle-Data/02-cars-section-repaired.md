# C13.2 — The `cars` Section, Repaired

> **The one-sentence version:** three rows in the shipped game are missing a comma, and the shift they
> cause manufactured a vehicle bug that does not exist — a worked example of a parse artefact
> masquerading as a finding.

[← C13.1 — handling.cfg](01-handling-cfg.md) · [Chapter 13 hub](C13-Vehicle-Data.md) ·
[Next: C13.3 — carcols & carmods →](03-carcols-and-carmods.md)

**Confidence:** ✅ Verified
**Closes:** [C11.1 §2](../C11-IDE-And-IPL/01-ide-definitions.md)

---

## 1. What C11.1 saw

[C11.1 §2](../C11-IDE-And-IPL/01-ide-definitions.md) reported the `cars` section as genuinely
variable-width — 15, 14, 12 and 11 fields — and quoted the file's own comment as the explanation:

```
# cars have two extra fields wheelmodelId and wheel scale
# planes have one extra field model id of low level of detail
```

That explanation is **half right**. Type does drive width — but two of the four widths were defects,
not design.

## 2. The real width distribution

After repairing the three malformed rows (§3):

| Type | Rows | Fields |
|---|---:|---:|
| `car` | 144 | 15 |
| `plane` | 13 | 15 (one at 11) |
| `heli` | 11 | 15 |
| `boat` | **10** | **11** |
| `bike` | 10 | 15 |
| `trailer` | 9 | 15 |
| `train` | 6 | 15 |
| `mtruck` | 5 | 15 |
| `bmx` | 3 | 15 |
| `quad` | 1 | 15 |

✅ *Verified.* **Only one type genuinely differs: `boat`, at 11 fields.** Boats carry neither the wheel
model nor the wheel scale, which is the file comment's rule working exactly as stated.

The 14-field rows were a defect. The 12-field row was a trailing comma. And one `plane` at 11 fields is
a real content quirk — the `skimmer`, a seaplane, written with the boat field set:

```
460, skimmer, skimmer, plane, SEAPLANE, SKIMMER, null, ignore, 5, 0, 0
```

🟡 *Reasoned:* a seaplane authored from a boat template and never widened. Its `HandlingId` is
`SEAPLANE`, and `handling.cfg` carries a `$ SEAPLANE` flying record
([C13.1 §2](01-handling-cfg.md)), so the vehicle is complete despite the short row.

## 3. The three malformed rows

In `vehicles.ide`, three rows separate the **model name** and **TXD name** with whitespace instead of a
comma:

| ID | Row as written |
|---:|---|
| 585 | `585,	emperor		emperor, 	car, 	EMPEROR, 	EMPEROR, ...` |
| 586 | `586,	wayfarer	wayfarer,	bike,	WAYFARER,	WAYFARE, ...` |
| 593 | `593,	dodo		dodo, 	plane, 	DODO,	 	DODO, ...` |

✅ *Verified.* Each yields **14 fields instead of 15**, and every column after the first shifts left by
one.

Note all three are consecutive-ish IDs in the same file and all three duplicate the name (`emperor
emperor`, `dodo dodo`) — the shape of a copy-paste template where the separator was lost once and
propagated.

## 4. ⚠️ The false anomaly

Here is what makes this page worth writing.

Parsing naively, the shifted columns put the *game name* where the **HandlingId** belongs. For row 586:

| Column | Naive read | Actual |
|---|---|---|
| HandlingId | `WAYFARE` | `WAYFARER` |
| Game name | `wayfarer` | `WAYFARE` |

`handling.cfg` contains `WAYFARER`. It does **not** contain `WAYFARE`.

So the first cross-check run reported:

> *"cars whose HandlingId has no handling.cfg entry: 1 — `WAYFARE`"*

which reads exactly like a genuine dangling reference in the shipped game. It is not. It is the comma
defect, seen through a parser that did not know about it.

**After repair: 212 / 212 HandlingIds resolve.** There is no missing handling entry in San Andreas.

## 5. Repairing it

The repair is safe because the defect has an unambiguous signature — a field containing internal
whitespace, sitting immediately before a known vehicle type:

```python
VEHICLE_TYPES = {'car','boat','train','heli','plane','mtruck','quad','bmx','trailer','bike'}

def parse_cars_row(line):
    f = [x.strip() for x in line.split(',')]
    while f and f[-1] == '':
        f.pop()                                   # trailing comma (squalo)
    # model and TXD name run together, so the type lands one column early
    if len(f) > 2 and f[2] in VEHICLE_TYPES and re.search(r'\s', f[1]):
        parts = re.split(r'\s+', f[1])
        if len(parts) == 2:
            f = [f[0], parts[0], parts[1]] + f[2:]
    return f
```

The `f[2] in VEHICLE_TYPES` guard is what makes it sound: it fires only when the type column is
demonstrably in the wrong place. Applied to all 212 rows it repairs exactly three and touches nothing
else.

## 6. The lesson

Three defects in 212 rows is a 1.4 % error rate in hand-maintained data — unremarkable. What matters is
what the defect *did* downstream.

A cross-check between two files found a discrepancy. The discrepancy was real in the sense that the two
files disagreed as parsed. But the disagreement was **manufactured by the parser**, and reporting it
would have put a non-existent bug into the encyclopedia with a ✅ next to it.

The check that caught it: **when a cross-reference fails for exactly one record, look at that record's
raw text before believing the failure.** One anomaly in 212 is far more likely to be a data-entry
oddity than a systematic engine fact — and the raw line showed the missing comma immediately.

This joins the running list of near-misses:

| Where | The trap |
|---|---|
| [C10.3 §2](../C10-2dEffect/03-the-light-record.md) | Off-by-one offset produced 11 plausible texture names |
| [C11.2 §5](../C11-IDE-And-IPL/02-ipl-placements.md) | A top-six table read as a distribution |
| [C12.1 §1.1](../C12-Path-Network/01-nodes-dat.md) | One file inspected, 63 others differed |
| **C13.2** | **A parse artefact reported as a game bug** |

---

### Key takeaways

- Only **one vehicle type genuinely has a different width**: `boat` at 11 fields, exactly as the file's
  own comment says. Everything else is 15.
- ⚠️ **Three retail rows (585 emperor, 586 wayfarer, 593 dodo) are missing a comma** between model name
  and TXD name, shifting every later column left by one.
- One `plane` — the `skimmer` — is written with the **boat field set** at 11 fields; 🟡 a seaplane from a
  boat template.
- ⚠️ **The defect manufactured a false bug**: a naive parse reports `WAYFARE` as a missing HandlingId.
  After repair, **212/212 resolve** — there is no such bug.
- The repair is safe because the signature is unambiguous: internal whitespace in field 1 with a known
  vehicle type in field 2.
- **When a cross-reference fails for exactly one record, read that record's raw text before believing
  it.** One-off failures are usually data entry, not engine facts.

**Continue:** [C13.3 — carcols & carmods](03-carcols-and-carmods.md)
