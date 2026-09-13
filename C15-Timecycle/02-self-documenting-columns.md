# C15.2 — The Columns the File Names Itself

> **The one-sentence version:** every other data file in this encyclopedia left its columns ⏳ open
> rather than borrow a community table — this one names them in a comment, which is a different
> evidence class and licenses naming them here.

[← C15.1 — Twenty-three weathers, eight hours](01-weathers-and-hours.md) ·
[Chapter 15 hub](C15-Timecycle.md) · [Next: C15.3 — The fifth defect →](03-the-fifth-defect.md)

**Confidence:** ✅ Verified (the header exists, verbatim) / 🟡 (the group→field mapping)

---

## 1. Why this page is different

[C13.1 §5](../C13-Vehicle-Data/01-handling-cfg.md) declined to name `handling.cfg`'s 36 columns.
[C14.3 §6](../C14-Peds-And-Weapons/03-stats-and-crosschecks.md) declined for six more files. The reason
was consistent: a community table is a **name accepted without asking what the bytes say**, and
adopting one reproduces the failure that produced the `0x253F2FE` correction in
[C10.1 §3](../C10-2dEffect/01-the-record-and-corrections.md).

`timecyc.dat` is the exception, because **the names come from the file**. Rockstar's own developers
wrote a column header into a comment at the top of each block:

```
//Amb			Amb_Obj	Dir			Sky top			Sky bot		SunCore		SunCorona
  SunSz	SprSz	SprBght	Shdw	LightShd	PoleShd	FarClp	FogSt	LightOnGround
  LowCloudsRGB	BottomCloudRGB	WaterRGBA			Alpha1	RGB1			Alpha2	RGB2	CloudAlpha
```

✅ *Verified* — reproduced verbatim from the shipped file.

That is primary-source evidence about the format, from the people who wrote it. It is a stronger basis
than any external table, and it is why this page names columns where others would not.

## 2. The twenty-four groups

Reading the header as a list of logical groups:

| # | Group | 🟡 Reading |
|---:|---|---|
| 1 | `Amb` | ambient light colour |
| 2 | `Amb_Obj` | ambient applied to objects |
| 3 | `Dir` | directional light colour |
| 4 | `Sky top` | sky gradient, upper |
| 5 | `Sky bot` | sky gradient, lower |
| 6 | `SunCore` | sun disc colour |
| 7 | `SunCorona` | sun corona colour |
| 8 | `SunSz` | sun size |
| 9 | `SprSz` | sprite size |
| 10 | `SprBght` | sprite brightness |
| 11 | `Shdw` | shadow strength |
| 12 | `LightShd` | light shadow |
| 13 | `PoleShd` | pole shadow |
| 14 | `FarClp` | far clip distance |
| 15 | `FogSt` | fog start |
| 16 | `LightOnGround` | ground light |
| 17 | `LowCloudsRGB` | low cloud colour |
| 18 | `BottomCloudRGB` | bottom cloud colour |
| 19 | `WaterRGBA` | water colour + alpha |
| 20 | `Alpha1` | — |
| 21 | `RGB1` | — |
| 22 | `Alpha2` | — |
| 23 | `RGB2` | — |
| 24 | `CloudAlpha` | cloud alpha |

The **names** are ✅ (they are in the file). The **readings** in the right column are 🟡 — expansions of
abbreviations, obvious in most cases (`SunSz` = sun size) and not in others (`Alpha1`/`RGB1` name a
pair whose purpose the header does not state).

## 3. Mapping 24 groups onto 51 fields

The header names 24 groups; a row carries 51 whitespace-separated numbers. Groups therefore span
multiple fields, and the arithmetic constrains the assignment:

```
Amb(3) Amb_Obj(3) Dir(3) SkyTop(3) SkyBot(3) SunCore(3) SunCorona(3)   = 21
SunSz(1) SprSz(1) SprBght(1) Shdw(1) LightShd(1) PoleShd(1)            = 27
FarClp(1) FogSt(1) LightOnGround(1)                                     = 30
LowCloudsRGB(3) BottomCloudRGB(3)                                       = 36
WaterRGBA(4)                                                            = 40
Alpha1(1) RGB1(3) Alpha2(1) RGB2(3)                                     = 48
CloudAlpha(1)                                                           = 49
```

That reaches **49, not 51** — so two fields are unaccounted for.

🟡 *Reasoned:* the seven colour triples and one RGBA are forced (a colour is 3 or 4 values), and the
scalar names are forced (a "size" is one number). The slack is in the middle block, where two of the
scalars are plausibly pairs. Looking at a real row, the eighth group reads `1.00 1.00 0.30 200 100 0` —
**six values where `SunSz` alone would be one**, which shows the header's spacing does not map
one-to-one onto groups.

⏳ **Open, honestly.** The header gives the vocabulary; it does **not** give the arity. Establishing
the exact boundaries needs either the parser in the executable or a correlation study — change one
value, observe which visual property moves. Neither was done here, and the arithmetic above is offered
as a starting hypothesis that is **known to be two short**.

**This is the right place to stop.** Naming the columns is supported by the file; assigning them widths
is not, and the difference matters.

## 4. What the values look like

From `EXTRASUNNY_LA` at Midnight:

```
22 22 22   220 212 130   255 255 255   0 23 24   0 31 32   255 128 0   5 0 0
1.00 1.00 0.30 200 100 0   400.00 100.00 1.00 30 20 0   3 3
```

✅ Every field in all 183 well-formed rows is numeric — verified. The mix is integers in 0–255 for
colours and floats for distances and scales, consistent with the header's vocabulary.

Note `400.00` and `800.00` in the far-clip position across blocks, and `100.00` for fog start — values
in world units that match the draw distances in
[C11.1 §1](../C11-IDE-And-IPL/01-ide-definitions.md)'s IDE rows. The timecycle and the object draw
distances are working in the same coordinate space.

## 5. The lesson about evidence classes

Three ways a column name can arrive:

| Source | Class | Used here? |
|---|---|---|
| The file's own comment | ✅ **primary** | **yes** — this page |
| Derived from behaviour or code | ✅ verified | where available |
| A community table | 🔷 external | **no** — see [X1 §1](../X1-SDK-Cross-Reference/X1-SDK-Cross-Reference.md) |

The distinction is not pedantry. A community table for `timecyc.dat` would very likely agree with the
header — but if it did not, the header wins, and knowing *which* source a name came from is what makes
that adjudication possible.

**The header is also incomplete**, per §3, and a borrowed table would have papered over exactly the gap
this page reports. That is the concrete cost of adopting names: it hides where the knowledge runs out.

---

### Key takeaways

- **`timecyc.dat` names its own columns** in a per-block comment — the only file in this encyclopedia
  that does.
- That is **primary-source evidence**, which is why this page names columns where
  [C13.1 §5](../C13-Vehicle-Data/01-handling-cfg.md) and
  [C14.3 §6](../C14-Peds-And-Weapons/03-stats-and-crosschecks.md) refused to.
- **24 named groups** — colours, sun and sprite parameters, fog and clip distances, cloud and water
  colours.
- ⏳ **The header gives vocabulary, not arity.** A forced-arity reconstruction reaches **49 of 51**
  fields, so two are unaccounted for — recorded as a hypothesis known to be short.
- Values are integers 0–255 for colours and floats for distances; **far-clip and fog values share the
  world-unit space** of IDE draw distances.
- A borrowed column table would have **hidden the two-field gap** — the concrete cost of adopting names
  instead of deriving them.

**Continue:** [C15.3 — The fifth defect](03-the-fifth-defect.md)
