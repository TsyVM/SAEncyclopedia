# C25.2 — The Census and Its Named Exceptions

> **The one-sentence version:** across 993 objects the enumerated fields stay inside the domains the header
> documents, but the continuous fields do not — a mass cap ignored 514 times, a buoyancy over 100 %, an
> out-of-range collision code, and one object marked breakable with no way to break — each exception named
> rather than rounded away.

[← C25.1 — The record format](01-the-record-format.md) · [Chapter 25 hub](C25-Object-Physics.md)

**Confidence:** ✅ Verified (the census, the enumerated domains, the named exceptions) / ⏳ (the physical
units of the scalars)

---

## 1. The population

The file yields **993 well-formed records** — 758 basic and 235 breakable — plus two section markers and two
truncated rows (§4). Everything below is a complete census over those 993, not a sample.

The five integer code fields honour the domains the header lists:

| Field | Values used (with counts) | Documented domain | In range? |
|---|---|---|:--:|
| `cdamage_effect` (I) | `0`×680, `200`×217, `20`×46, `1`×46, `202`×6 | `0 1 20 21 200 202` | ✅ (`21` unused) |
| `causes_explosion` (L) | `0`×985, `1`×10 | boolean | ✅ |
| `fx_type` | `0`×931, `2`×63, `1`×1 | `0 1 2 3` | ✅ (`3` unused) |
| `break_gun_mode` | `1`×145, `0`×49, `2`×41 | `0 1 2` | ✅ |
| `break_sparks` | `0`×222, `1`×13 | boolean | ✅ |

The FX system is off for almost everything: `fx_name` is `"none"` on 928 of 993 records, and the eight
distinct FX names that do appear are explosion and tree-hit effects. Only 74 objects play an effect when hit
or destroyed — the rest are inert scenery.

✅ *Verified:* every enumerated field stays inside its documented domain.

## 2. Where the data overflows its own header — named

The continuous fields are a different story. The header states ranges for them, and the data ignores several
— always in a way that is deliberate, never random, and always nameable:

**Mass — a 50,000 kg cap ignored 514 times.** The header says `[kilograms 1 to 50000]`. In fact **514** of
the 993 objects exceed 50,000, almost all sitting at exactly `99999.0`. That value is not a mass anyone
measured — it is the file's de-facto "immovable" sentinel, applied to lamp posts, gun turrets and anything
the physics should treat as effectively pinned. The documented cap is aspirational; the real convention is
"99999 means don't move".

**Percent submerged — one value over 100 %.** The header allows `[10 to 120]`. Exactly one object, `dump1`,
carries **150**, meaning it is authored to sit lower in water than "fully submerged" nominally allows. A
single deliberate overflow, recorded rather than clamped.

**Special collision response — one out-of-range code.** `special_cdr` (J) is documented `0…9`. One object,
`imy_bbox`, carries **20**, a value with no documented meaning; the engine will read it and fall through its
switch to the default. A lone typo-or-leftover, named.

**Camera-avoid — an undocumented third value.** The header says `(0)` for no and `(1)` for yes. Two objects,
`portakabin` and `CARRIER_DOOR_SFSe`, carry **2**. Whatever the third value was meant to select, the header
never mentions it, and only these two use it.

⚠️ *Named exceptions, verified and reproducible:* the mass cap (514×), `dump1`'s 150 % buoyancy, `imy_bbox`'s
`J = 20`, and `portakabin` / `CARRIER_DOOR_SFSe`'s `camera = 2`.

## 3. The breakable-without-params defect

The sharpest defect is the one [C25.1 §4](01-the-record-format.md) already flagged from the parser side.
`sec_keypad` sets `cdamage_effect = 200` — *breakable* — but supplies only 17 fields, so it has **no** break
velocities, no gun-break mode, nothing. The header's own instruction ("needs set if I ≥ 200") is unmet.
Because the parser attaches break data by field count rather than by the collision code, the engine does not
notice: it registers `sec_keypad` as a 17-field basic object and the `I = 200` simply has no break behaviour
behind it. It is a data bug that is invisible to the loader — precisely the kind the field-count design
makes possible.

This mirrors the [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md) philosophy that a shipped defect is worth
recording as carefully as a correct value: it tells you how the pipeline actually behaved, not how it was
meant to.

## 4. Two truncated rows

Two data lines — `JUD_LAN` and `spraydoor_LAw2` — carry only **13** fields: name, the seven physics floats,
and the five integer codes, but no FX offset and no FX name. `sscanf` fills 13 and returns; the loader keeps
them. Both set `fx_type = 0` (no FX system), so the missing FX offset and name are never consulted, and the
truncation is harmless in practice — but it is a truncation, and the format expects at least 17 fields, so it
is recorded as a defect rather than treated as a third record variant. (They are not counted among the 993
well-formed records above.)

## 5. What the census does not settle

⏳ The **physical units and dynamics** of the scalar fields are not derived here. The header labels mass in
kilograms, air resistance and elasticity as 0–1 scales, uproot limit as a force magnitude, and the census
confirms the value *distributions* are consistent with those labels — air resistance clusters near 1.0
(low drag), elasticity stays low (little bounce), buoyancy sits at 50 % for most objects. But how each
number enters the rigid-body integration is engine behaviour for a physics chapter, not this format one.

The **grammar, the field identities, the two variants, the census and the enumerated domains are ✅**; the
quantitative interpretation of the continuous fields is the open remainder.

---

### Key takeaways

- ✅ Census of **993** objects (758 basic + 235 breakable); every **enumerated** field
  (`cdamage_effect`, `fx_type`, `causes_explosion`, `break_gun_mode`, `break_sparks`) stays inside its
  documented domain.
- ⚠️ The **continuous** fields overflow the header's ranges in named ways: **514** objects exceed the 50,000
  kg mass cap (`99999` is the de-facto "immovable" sentinel); `dump1` has 150 % buoyancy; `imy_bbox` has an
  out-of-range `special_cdr = 20`; `portakabin` and `CARRIER_DOOR_SFSe` use an undocumented `camera = 2`.
- ⚠️ `sec_keypad` is **declared breakable (`I = 200`) with no break parameters** — a defect the field-count
  parser cannot detect.
- ⚠️ `JUD_LAN` and `spraydoor_LAw2` are **truncated** to 13 fields; harmless (`fx_type = 0`) but recorded.
- ⏳ The physical **units** of the scalar fields are left to the physics model.

**Continue:** [Chapter 25 hub](C25-Object-Physics.md)
